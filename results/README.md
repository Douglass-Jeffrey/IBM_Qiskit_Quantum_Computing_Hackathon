# Results

Every table the project produced, grouped by the question it answers. Errors are in debye (D).

- **Naming:** `nbNN_*` files come from notebook NN (development data only: `dev`, `dev_unseen` and cross-validation
  inside the training sets); `test_runNN_*` files come from evaluation run NN of `scripts/final_eval.py` (the test
  sets); `hardware_*` files from `scripts/hardware_run.py`.
- **Provenance:** each `.csv` has a `.meta.json` file next to it recording the commit, configuration, seeds and
  package versions it was made with. (Files made before the notebooks were renumbered record the earlier notebook
  names; see [notebooks/README.md](../notebooks/README.md#earlier-names).)
- **Run log:** [test_runs.csv](test_runs.csv) has one row per evaluation run: what it was, its commit and config hash.
- **Per-model summary:** [model_summary.md](model_summary.md) describes every quantum model and its same-size
  classical counterpart, with test MAE and RMSE at every training size ([model_summary.csv](model_summary.csv) holds
  the same numbers).

Figures are grouped the same way in [../figures/](../figures/README.md). The code finds every file by its name
(`qm9dipole.provenance.results_file`).

## [comparison/](comparison/): quantum models vs classical models of the same size

Start here. Quantum kernel ridge (three encoders), a no-entanglement control, the projected quantum kernel and the
team's ⟨Z⟩ ridge, each against classical models with the same inputs, training sets and tuning budget (N ≤ 1,000).

| File | What it holds |
|---|---|
| `test_run01_headline.csv` | **The headline table**: the quantum model chosen by CV vs its RBF twin (paired difference), the best quantum-comparable classical model, the best classical model at that N, and the mean baseline; per test set and N |
| `test_run01_quantum.csv` | Every quantum and same-size classical model × input set × N × seed: MAE, RMSE, R² and bias on both test sets and both development sets, with the CV score and the settings chosen |
| `test_run02_quantum.csv` | The shot-aware follow-up: quantum kernels tuned and evaluated at 100 / 1,000 / 10,000 shots, plus the exact models and RBF twins again |
| `test_run02_shot_aware.csv` | The same shot budget, tuned on exact kernels (run 1) vs with the shots inside cross-validation (run 2) |
| `nb08_main.csv` | Development sets: every quantum model vs its classical twins on the same 10 PLS inputs |
| `nb08_headline.csv` | Development sets: the headline comparison |
| `nb08_paired.csv` | Paired per-seed differences, quantum minus RBF kernel ridge on the same inputs |
| `nb08_qubits.csv` | Error vs number of inputs/qubits (k = 4–16), quantum vs classical |
| `nb08_inputs.csv` | Input variants (PLS, PCA, top-k, composition only), quantum vs classical |
| `nb08_extended.csv` | Quantum kernel ridge vs RBF kernel ridge up to N = 10,000 (simulator) |
| `nb08_kernel_diagnostics.csv`, `nb08_concentration.csv` | Kernel spectra, effective parameters, kernel–target alignment; exponential concentration vs qubits and angle scale |
| `nb08_invariance.csv` | End-to-end rotation, translation and relabelling test of the quantum model, with a negative control |
| `nb06_track_b.csv`, `nb06_quantum_baselines.csv` | The quantum-comparable classical baselines (Track B: mean, ridge, RBF kernel ridge, random forest, XGBoost on three input sets) |

## [hardware/](hardware/): finite shots, noise, cost and IBM hardware

| File | What it holds |
|---|---|
| `hardware_ibm_quebec_20261007-1925_scores.csv` | **IBM `ibm_quebec` (run 4)**: the team's ⟨Z⟩ ridge on the device vs exact and ideal-device simulation |
| `hardware_ibm_quebec_20261007-1925_ibm_record.json` | IBM's record of the job: backend, timestamps, execution span, 45 s of QPU time charged |
| `hardware_ibm_quebec_20261007-1925_raw_bits.npz`, `..._features.npz` | The device's raw shot outcomes (160 circuits × 1,000 shots) and the ⟨Z⟩ values computed from them |
| `hardware_ibm_quebec_20261007-1925_job.json`, `..._plan.csv` | The submission record (molecules, settings) and the circuit plan |
| `hardware_FakeQuebec_20261007-2019_*` | Run 3: the same circuits (plus the projected kernel and a fidelity-kernel block) on the FakeQuebec noise model |
| `test_run01_shots.csv`, `test_run01_shots_summary.csv` | MAE vs shots (100 / 1,000 / 10,000, 5 repetitions) for quantum kernel ridge, the projected kernel and the ⟨Z⟩ ridge, tuned on exact kernels |
| `test_run01_noise.csv`, `test_run01_noise_kernels.npz` | FakeFez noise on 60 test molecules (exact, shots only, noisy, corrected) and the kernels |
| `nb09_circuit_costs.csv`, `nb09_model_costs.csv` | Circuit sizes transpiled for IBM Heron; circuits and QPU time per model and N |
| `nb09_shots.csv`, `nb09_shot_validation.csv`, `nb09_noise.csv` | Development sets: shots, the shot model checked against Aer, FakeFez noise |

## [classical/](classical/): the most accurate classical models (Track A)

| File | What it holds |
|---|---|
| `test_run01_track_a.csv` | Ridge, RBF kernel ridge, XGBoost and the latent-charge network from N = 100 to 99,198, on the test sets |
| `nb06_track_a.csv` | The same on the development sets (3 seeds at every N) |
| `nb06_charge_selection.csv` | Architecture search for the charge network (inner validation split) |
| `nb06_effective_df.csv` | Effective number of parameters of each model |
| `nb07_*.csv` | Formula-grouped CV, permutation importance, end-to-end invariance of the charge network |

## [development/](development/): data, features and preprocessing

| File | What it holds |
|---|---|
| `nb02_descriptor_invariance.csv` | Rotation, translation and relabelling test of the first descriptors, with a negative control |
| `nb03_recommendations.csv` | Field-by-field data analysis: what to keep and how to treat it |
| `nb04_feature_catalog.csv` | Every feature: block, level (molecule or atom), whether it is a function of Z and R, kept or dropped and why |
| `nb05_preprocessing.json` | The chosen scaling, target transform and the PLS(10) compression for the quantum-comparable track |
| `nb05_scaling.csv`, `nb05_target.csv`, `nb05_compression_cost.csv` | The evidence for those choices |
