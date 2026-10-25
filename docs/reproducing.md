# Reproducing the results

Everything runs on an ordinary laptop CPU; runtimes below are for an 8-core laptop. Only the optional hardware run
needs an IBM Quantum account.

## 1. Install

**Requirements.** Python **3.12**. Exact pins are in [`requirements.txt`](../requirements.txt), compatible ranges in
[`pyproject.toml`](../pyproject.toml). Main packages: qiskit 2.5.2, qiskit-aer 0.17.2, qiskit-ibm-runtime 0.50.0,
numpy 2.5.3, scipy 1.18.1, scikit-learn 1.9.1, pandas 3.0.6, xgboost 3.4.1, pytest 9.1.1.

From the repository root:

```bash
# Option A: conda
conda env create -f environment.yml
conda activate qm9dipole
pip install -r requirements.txt        # after activation; also installs this package in editable mode

# Option B: plain Python 3.12
python3.12 -m venv .venv               # Windows: py -3.12 -m venv .venv
source .venv/bin/activate              # Windows: .venv\Scripts\activate
pip install -r requirements.txt

pip check && pytest -q                 # 671 tests; data-dependent ones skip until QM9 is downloaded
```

<details><summary>Troubleshooting</summary>

- **`No matching distribution found for numpy==2.5.3`:** the environment is not Python 3.12. Recreate it with 3.12.
- **Windows, `OSError` or `FileNotFoundError` with a very long path** (during `pip install`, or when a long run
  writes its fit cache under `data/processed/cache/`): the path exceeds Windows' 260-character limit. Clone into a
  shorter folder (the repository root should be under ~90 characters), or enable long paths (`LongPathsEnabled`).
- **The first notebook cell says the package belongs to another checkout:** the selected kernel's environment was
  installed from a different copy of the project; select this copy's environment.
- **Why `pip install` comes after `conda activate`:** activation sets `PYTHONNOUSERSITE=1`, which hides per-user
  packages; installing before activation can leave the environment broken once activated.
</details>

**Notebook kernels.** The notebooks use the standard `python3` kernel. In VS Code, pick the `qm9dipole` (conda) or
`.venv` environment in the kernel picker; in JupyterLab, start `jupyter lab` from the activated environment. The
first cell of every notebook checks the environment and warns about version differences.

## 2. Get the data

The QM9 data is not redistributed here. [Notebook 00](../notebooks/00_setup_and_download.ipynb) downloads the three
files (86 MB) into `data/raw/` and verifies them against figshare's published MD5 checksums. To download by hand,
fetch figshare files 3195389 (`dsgdb9nsd.xyz.tar.bz2`), 3195404 (`uncharacterized.txt`) and 3195392 (`readme.txt`)
from collection 978904 into `data/raw/`; their sizes and checksums are in
[`src/qm9dipole/data.py`](../src/qm9dipole/data.py).

## 3. Quick checks

```bash
pytest -q                                          # unit tests, including the invariance and split checks
python scripts/final_eval.py --dry-run --smoke     # the whole evaluation pipeline on development data, ~10 min
```

`QM9DIPOLE_SMOKE=1` makes notebooks 08 and 09 run a minutes-long version (one seed, N = 100, small axes).

## 4. Reproduce the headline comparison (≈ 45 minutes)

This retrains every model of the quantum-vs-classical comparison (22 models × 3 training sizes × 3 seeds) and scores
the test sets:

```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/00_setup_and_download.ipynb      # download, ~1 min
jupyter nbconvert --to notebook --execute --inplace notebooks/01_parse_and_split.ipynb         # parse + splits, ~1 min
jupyter nbconvert --to notebook --execute --inplace notebooks/04_cleaning_and_features.ipynb   # features, ~2 min
python scripts/final_eval.py --sections quantum --allow-dirty                                  # ~40 min
```

- **What the last command writes:** `results/comparison/test_runNN_quantum.csv` (the next free run number) and a row
  in `results/test_runs.csv`.
- **Why `--allow-dirty`:** the script normally refuses to run with uncommitted changes (so every logged run maps to
  a commit), and executing the notebooks above modifies them.
- **No re-tuning:** the scaling, compression and model grids come from the committed results of the development
  notebooks; every model is tuned by cross-validation inside its own training sets, exactly as in the original run.

To compare with the committed run (the difference is 0.0 on the same platform):

```python
import glob, pandas as pd
new = sorted(glob.glob("results/comparison/test_run*_quantum.csv"))[-1]
key = ["model", "inputs", "seed", "n_train", "eval_set"]
a, b = (pd.read_csv(f).fillna({"inputs": "—"}).set_index(key)["mae_D"] for f in (new, "results/comparison/test_run01_quantum.csv"))
print(new, "max |Δ MAE| =", (a - b.loc[a.index]).abs().max())
```

## 5. The full pipeline, in order

| Step | What it does | Time |
|---|---|---|
| [00_setup_and_download](../notebooks/00_setup_and_download.ipynb) | Download QM9, verify checksums | ~1 min |
| [01_parse_and_split](../notebooks/01_parse_and_split.ipynb) | Parse, exclusions, splits (reproduces `splits/*.json` exactly) | ~1 min |
| [02_invariant_descriptors](../notebooks/02_invariant_descriptors.ipynb) | Descriptors; rotation, translation and relabelling tests | ~10 s |
| [03_exploratory_data_analysis](../notebooks/03_exploratory_data_analysis.ipynb) | Every QM9 field, on the training pool | ~4 min |
| [04_cleaning_and_features](../notebooks/04_cleaning_and_features.ipynb) | 188 molecule features, 85 per-atom features | ~3 min |
| [05_scaling_and_compression](../notebooks/05_scaling_and_compression.ipynb) | Scaling and target transform per track; PLS to 10 inputs | ~1.5 h |
| [06_classical_models](../notebooks/06_classical_models.ipynb) | Track A to 99,198 labels; Track B baselines | ~8 h |
| [07_generalization_and_diagnostics](../notebooks/07_generalization_and_diagnostics.ipynb) | New formulas, errors, importance, invariance | ~10 min |
| [08_quantum_vs_classical](../notebooks/08_quantum_vs_classical.ipynb) | Quantum models vs matched classical; qubits, inputs, kernel diagnostics | ~2 h |
| [09_shots_noise_and_cost](../notebooks/09_shots_noise_and_cost.ipynb) | Quantum cost, finite shots, simulated noise | ~20 min |
| `scripts/final_eval.py` | Retrain everything; score the test sets (runs 1 and 2) | ~4.5 h; ~20 min |
| `scripts/hardware_run.py` | Hardware circuits on a noise model or IBM hardware (runs 3 and 4) | ~2 min; 45 s QPU |
| [10_test_set_results](../notebooks/10_test_set_results.ipynb) | Presents every evaluation run | ~3 min |
| [11_quantum_feature_ridge](../notebooks/11_quantum_feature_ridge.ipynb) | The team's ⟨Z⟩ ridge on its own, from the saved results | seconds |

- **Run them in order:** each notebook reads the previous ones' outputs. Notebooks 03–09 never read a test set.
- **Caching:** the long notebooks cache finished fits per commit in `data/processed/cache/`, so an interrupted run
  resumes where it stopped.
- **Headless:** `jupyter nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=-1 notebooks/<name>.ipynb`.

```bash
python scripts/final_eval.py                                              # main evaluation (configs/test_eval.yaml), ~4.5 h
python scripts/final_eval.py --config configs/test_eval_shot_aware.yaml   # shot-aware follow-up, ~20 min
python scripts/hardware_run.py                                            # hardware circuits on the FakeQuebec noise model (local)
python scripts/hardware_run.py --backend ibm_quebec                       # plan only: circuits, shots, estimated QPU time
python scripts/hardware_run.py --backend ibm_quebec --parts qridge --submit --max-seconds 120 --account <name>
python scripts/hardware_run.py --retrieve <job id> --account <name>       # fetch and score a submitted job
```

**IBM account.** The hardware script reads an account saved once with
`QiskitRuntimeService.save_account(name="<name>", ...)` and selected with `--account <name>`. No token is stored in
this repository. Without `--submit` the script only prints the plan; with it, it shows the plan, asks for a typed
confirmation and caps the job at `--max-seconds` of QPU time.

## 6. Seeds and determinism

- **Splits:** split seed 2026. **Training sets:** seeds 0, 1, 2 (fill order, CV folds, XGBoost search, network
  initialization).
- **Simulated shots** are seeded deterministically (by seed, N, shots and repetition), Aer sampler seeds are fixed,
  and transpilation uses `seed_transpiler=0`, so simulated results reproduce exactly on the same platform. The
  hardware run's test molecules are drawn with seed 2027.
- **Provenance:** every results CSV has a `.meta.json` file with the commit, configuration, seeds and package
  versions. The commit hash ends in `-dirty` if code was uncommitted when the file was written.

## 7. Verification: a full from-scratch rerun

On 2026-10-24 the whole pipeline was rerun on a fresh clone, starting from the three raw QM9 files with no parsed
tables, features or cached fits: notebooks 00–11, both evaluation configs (`test_eval.yaml`,
`test_eval_shot_aware.yaml`) and the local noise-model hardware run, about 16.5 hours on an 8-core laptop.

- **Every reported number reproduced exactly** (difference 0.0): the splits and the feature table; every result of
  notebooks 03, 04 and 06–09; and every row of evaluation runs 1–3 (all 612 rows of the quantum-vs-classical
  comparison, all 316 rows of the classical learning curves, the shots, noise and shot-aware sections, and the
  noise-model hardware run). Only timings and run numbers differed.
- **The IBM hardware result** (run 4) cannot be rerun without QPU time, but recomputing it from the job's saved device
  measurements gives the same scores exactly (0.778 D on `ibm_quebec`).
- **Found and fixed:** PCA, used only in the input comparisons of notebooks 05 and 08, was not deterministic at
  N = 1,000 (scikit-learn's randomized solver, unseeded; up to 3 × 10⁻⁴ D). It now uses the exact solver, and those
  two notebooks were rerun. The outputs of notebooks 02, 03 and 05 were out of date and have been regenerated;
  notebook 05 now also records that its selection rule picks 8 inputs while the project keeps 10.

The committed notebooks, results and figures of notebooks 00–09 and 11 are the outputs of this rerun; evaluation
runs 1–4 are the original ones, which the rerun reproduced.
