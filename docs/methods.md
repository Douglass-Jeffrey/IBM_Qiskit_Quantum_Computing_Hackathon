# Methods

How the project works, from raw data to a job on IBM hardware. The [README](../README.md) gives the story and the
headline numbers; [results.md](results.md) gives every result; this page explains how they were produced. Terms are
defined at first use and collected in the [glossary](#glossary) at the end.

**Contents**

1. [The task](#1-the-task)
2. [Data](#2-data)
3. [Splits: familiar and unseen formulas](#3-splits-familiar-and-unseen-formulas)
4. [Features from atoms and geometry](#4-features-from-atoms-and-geometry)
5. [Two classical references](#5-two-classical-references)
6. [Quantum models](#6-quantum-models)
7. [Fair-comparison protocol](#7-fair-comparison-protocol)
8. [From exact simulation to real hardware](#8-from-exact-simulation-to-real-hardware)
9. [Evaluation runs](#9-evaluation-runs)
10. [Seeds and settings](#10-seeds-and-settings)
11. [Glossary](#glossary)

---

## 1. The task

**Target.** The dipole moment μ of a molecule measures how far apart its centres of positive and negative charge
are. We predict its magnitude |μ| in **debye (D)**; 1 D ≈ 0.21 e·Å (the dipole of one electron charge separated from
an equal positive charge by 0.21 ångström). A water molecule has |μ| ≈ 1.85 D; across QM9 the mean is 2.71 D, the
median 2.52 D, and a few charge-separated molecules (zwitterions) reach 29.6 D. Polarity drives solubility,
intermolecular forces and spectroscopy, so it is a property chemists care about.

**Inputs.** Each molecule is given as its atoms' **atomic numbers Z** and their **3D coordinates R** (in ångström),
and nothing else. QM9 also stores other computed properties (partial charges, orbital energies, polarizability…),
but they come from the same quantum-chemistry calculation that produced μ: a model that needs them can only be used
after that calculation has already been run, at which point μ is known. They were used only for exploration and
interpretation, never as model inputs.

**Why "fewer labels".** Every label in QM9 is the output of a density-functional-theory (DFT) calculation, which is
expensive. The question that motivated the project is therefore how prediction error falls as the number of
labelled training molecules N grows, and whether quantum models, which are often proposed for small-data problems,
do better at small N.

**Physical requirement.** |μ| does not change when a molecule is rotated, translated, or has its atoms listed in a
different order, so the models must not change either (they must be **invariant**).

## 2. Data

**Source.** QM9 (Ramakrishnan et al., *Scientific Data* 2014): 133,885 small organic molecules with up to nine
heavy atoms (C, N, O, F; plus H), from the GDB-17 enumeration. Geometries and properties were computed with DFT at
the B3LYP/6-31G(2df,p) level, so they are reference calculations, not experiments.

- **Download and verification** ([notebook 00](../notebooks/00_setup_and_download.ipynb),
  [`data.py`](../src/qm9dipole/data.py)): three files from figshare collection 978904 (86 MB in total), each checked
  against figshare's published size and MD5 checksum. The data is not redistributed in this repository.
- **Parsing** ([notebook 01](../notebooks/01_parse_and_split.ipynb)): the 133,885 `.xyz` records are streamed
  straight out of the compressed archive (never extracted) into one table with every field kept; 0 parse failures.

**Exclusions** (3,202 molecules removed, **130,683 kept**):

| Excluded | Count | Reason |
|---|---|---|
| Listed in QM9's `uncharacterized.txt` | 3,054 | The optimized geometry no longer corresponds to the intended molecule |
| Flagged in QM9's readme as hard to converge | 8 | Saddle points or loose convergence (11 listed; 3 were already excluded) |
| Duplicate entries | 140 | The same molecule listed twice (same formula and geometry); one copy is kept, so no molecule can sit in two sets |

Duplicates are found from the geometry (formula plus Coulomb-matrix spectrum, §4), not from InChI strings: standard
InChI merges tautomers, which are different molecules with dipoles up to 8.5 D apart.

**Data quality** ([notebook 03](../notebooks/03_exploratory_data_analysis.ipynb)). On the training pool: no missing
values, every molecule neutral, no imaginary vibrational frequencies, and every internal consistency rule from the
QM9 readme holds. The stored coordinate frames are arbitrary (only 1.6% are aligned with the principal axes), which
confirms that raw coordinates cannot be used directly. The 223 molecules at |μ| ≥ 9 D (0.22%) are valid values and
are kept; they weigh heavily on RMSE, which is one reason MAE is the primary metric.

## 3. Splits: familiar and unseen formulas

The brief asked whether models work on **molecular formulas absent from training** (e.g. a model trained without any
C₆H₆O₂ isomer, asked about one). The splits therefore hold out whole formulas as well as random molecules
([`splits.py`](../src/qm9dipole/splits.py), [`configs/splits.yaml`](../configs/splits.yaml), split seed 2026; the
molecule IDs of every set are saved in [`splits/`](../splits/)):

| Set | Molecules | Formulas | What it is for |
|---|---|---|---|
| Training pool | 99,198 | 497 | Every training set is drawn from it |
| `dev` | 5,000 | 257 | Development: model selection and diagnostics (familiar formulas) |
| `dev_unseen` | 4,183 | 26 | Development: whole formulas absent from training |
| **`test_familiar`** | 5,484 | 257 | Final evaluation, formulas present in training |
| **`test_unseen`** | 16,818 | 93 | Final evaluation, whole formulas absent from all training and development data |
| `test_familiar_q`, `test_unseen_q` | 80, 370 | 20, 93 | Small test subsets for the expensive shot, noise and hardware experiments |

- **Development vs test.** All model building used the training pool and the two development sets. The test sets
  were scored only by the evaluation script ([§9](#9-evaluation-runs)).
- **Nested training sets.** For each seed s ∈ {0, 1, 2}, one random fill order of the pool defines
  S(s, N) = its first N molecules, for N = 100, 300, 1,000, 3,000, 10,000, 30,000 and 99,198 (the whole pool). The
  sets are nested (S(s, 100) ⊂ S(s, 300) ⊂ …) and identical for every model, so every comparison is paired.
- **Anchors.** The 20 formulas of `test_familiar_q` are placed first in every fill order, so even the 100-molecule
  training sets contain them and those test molecules really are "familiar" at every N.
- **Checks.** Every property above (disjoint sets, held-out formulas absent from training, nesting, anchors, exact
  sizes, no excluded molecule) is asserted when the splits are built and unit-tested in
  [`tests/test_splits.py`](../tests/test_splits.py), each test paired with a negative control that breaks it.

## 4. Features from atoms and geometry

Models need fixed-length numeric inputs, and those inputs must be invariant (§1). Every feature is computed from Z
and R alone, plus tabulated constants such as covalent radii and electronegativities
([`descriptors.py`](../src/qm9dipole/descriptors.py), [`features.py`](../src/qm9dipole/features.py),
[`atoms.py`](../src/qm9dipole/atoms.py); [notebook 04](../notebooks/04_cleaning_and_features.ipynb)).

**Molecule-level features (188 after cleaning)**, used by every model except the charge network:

| Block | Features | What it encodes |
|---|---|---|
| Composition | 5 | Counts of C, H, N, O, F |
| Coulomb-matrix spectrum | 29 | Sorted eigenvalues of the Coulomb matrix (Rupp et al. 2012), whose entries are the nuclear repulsions Z_i Z_j / r_ij: geometry in an invariant form |
| Engineered geometry | 26 | Size and shape (gyration tensor, moments of inertia), bond counts and bond orders inferred from bond lengths, rings, and simple bond-dipole estimates |
| Functional groups | 19 | Atom types and charged groups read from the bond graph (ammonium, carboxylate, nitro-like, zwitterions…) |
| Charge-equilibration dipole | 3 | A no-fit physics estimate (QEq; Rappé & Goddard 1991): partial charges from tabulated electronegativities balanced against Coulomb interactions on the actual geometry, then their dipole |
| Radial distributions | 106 | For each element pair, how many atom pairs sit at each distance |

**Per-atom features (85)** describe each atom's local environment: element, bonded neighbours, bond orders, atoms two
and three bonds away, rings, radial and directional environment, and its QEq charge. They feed the latent-charge
network (§5).

**Cleaning** is unsupervised (it never looks at |μ|): zero-variance columns are dropped, and of two columns that carry
the same information up to a monotone transform (|Spearman ρ| ≥ 0.99), the less interpretable one is dropped.

**Invariance tests.** Every feature is recomputed after a random rotation, translation and atom relabelling of 100
molecules. Features change by at most 10⁻¹² (floating-point rounding); a deliberately non-invariant **negative
control** (the raw coordinates) changes by several ångström, which shows that the test can fail
([notebook 02](../notebooks/02_invariant_descriptors.ipynb), [`tests/test_invariance.py`](../tests/test_invariance.py)).
The trained models are tested end to end the same way (notebooks 07 and 08).

## 5. Two classical references

"The best classical model" and "the classical model a quantum model should be compared with" are different
questions, so the project builds two classical references:

| | **Track A**: best classical | **Track B**: quantum-matched classical |
|---|---|---|
| Question | How accurate can a classical model get from Z and R? | Given exactly what a quantum model gets, does the quantum model do better? |
| Training sizes | N = 100 up to the full 99,198-molecule pool | N = 100, 300, 1,000 (what a quantum kernel can afford) |
| Inputs | All 188 features, or the 85 per-atom features (charge network) | The quantum models' inputs: 10 PLS components of the 188 features, or composition only |
| Models | Ridge, RBF kernel ridge, XGBoost, latent-charge network | Mean, ridge, RBF kernel ridge, random forest (+ XGBoost) |
| Preprocessing | Chosen for accuracy at large N: log-then-standardize scaling, standardized target | Identical to the quantum models: Yeo-Johnson scaling, √\|μ\| target |
| Tuning | 5-fold CV up to N = 1,000, an inner validation split above | 5-fold CV, the same folds as the quantum models |

Every quantum result is reported against both: Track B is the like-for-like comparison; Track A shows how far
classical methods go without the quantum models' limits.

**Preprocessing choices** ([notebook 05](../notebooks/05_scaling_and_compression.ipynb)), each made by
cross-validation (CV) inside the training sets with a rule fixed before the comparison was run:
- **Feature scaling.** Raw features differ in scale by a factor of over a million. Track B uses a fitted
  Yeo-Johnson power transform (it leaves the least skew and the fewest extreme values, which matters because quantum
  encodings turn values into rotation angles that wrap around every 2π). Track A uses log-then-standardize.
- **Outlier guard.** Every unbounded scaling clips scaled values at ±10. Without it, a new molecule with a value far
  outside a small training set's range (e.g. a radial-distribution bin that is almost always empty) can be scaled to
  10⁷ and make a linear model predict 10⁵ D. The clip only acts on such outliers.
- **Target transform.** Track B models predict √|μ| (better for the kernel models); Track A predicts standardized |μ|.
  Every prediction is transformed back to debye before scoring, and negative predictions are clipped at 0 D.
- **Compression to 10 inputs.** A quantum model takes one input per qubit, so 188 features must be compressed.
  Partial least squares (PLS; the directions of the inputs with the largest covariance with the target, refit on
  every training set) beat PCA and top-k feature selection by mutual information. The models use **10
  components**. The one-standard-error rule at N = 1,000 (the smallest k whose CV error is within one standard error
  of the best) gives 8 with the ±10 clip in place and gave 10 before it; 8 and 10 are a tie (CV MAE 0.500 vs 0.498 D,
  against a seed-to-seed spread of about 0.02 D), so the number is fixed at 10 in `configs/explore.yaml`.

**Classical models** ([`models/classical.py`](../src/qm9dipole/models/classical.py),
[`models/fitters.py`](../src/qm9dipole/models/fitters.py)):
- **Ridge** and **RBF kernel ridge** (exact kernel ridge, so N ≤ 10,000), with grids of penalty α and bandwidth.
- **Random forest** and **XGBoost** (gradient-boosted trees; 20-draw randomized hyperparameter search with early
  stopping).
- **Latent-charge network** ([`models/charge.py`](../src/qm9dipole/models/charge.py), pure NumPy). A dipole is a sum
  over atoms, so the model predicts one partial charge qᵢ and a small bond-directed atomic dipole for every atom
  from its environment, refined by three rounds of message passing along the bonds, and returns
  |Σ qᵢ rᵢ + Σ atomic dipoles|. Charges sum to zero, so the result is translation-invariant; invariant charges times
  rotating positions give a rotating dipole vector with an invariant length. The model is therefore invariant **by
  construction**, and it is trained on |μ| only (it never sees a reference charge). 118,346 parameters; MAE loss,
  AdamW, early stopping on an inner 10% validation split. Its architecture (three message-passing rounds with atomic
  dipoles) was chosen among eight candidates on that inner validation split of S(0, 10,000).

## 6. Quantum models

All quantum models share one idea: a molecule's 10 input numbers x become **rotation angles** γ·x of a fixed circuit
U(x). Running U(x) on the all-zeros state prepares an n-qubit quantum state |ψ(x)⟩, a unit vector of 2ⁿ complex
amplitudes. **Nothing in the circuit is trained**; γ (the angle scale) is a hyperparameter, like an RBF bandwidth. The
circuits are Qiskit `QuantumCircuit`s built by the team's encoder, `build_encoding_circuit`
([`models/quantum.py`](../src/qm9dipole/models/quantum.py)).

**Encoders (circuits U(x))**
- **ZZ feature map** (Qiskit's `zz_feature_map`, Havlíček et al. 2019): Hadamards, phase rotations P(2γxᵢ), and for
  each neighbouring pair of qubits a CX–P–CX block that writes the *product* of two inputs into the phase. Two passes,
  10 qubits. This is the encoder of the headline model.
- **RY + CZ**: one RY(γxᵢ) rotation per input, a fixed RX(π/4) on each qubit, and a chain of CZ gates between passes.
- **RY-RZ**: two inputs per qubit (RY then RZ), so 10 inputs fit on 5 qubits.
- **Product-state control**: RY with a single pass, so no entangling gate ever acts. Its kernel is exactly the
  classical kernel Π cos²(γΔxᵢ/2). If an entangled kernel cannot beat it, entanglement contributed nothing.

**Readouts: three ways to turn states into predictions**

```
Z, R  →  188 invariant features  →  Yeo-Johnson scaling (clipped at ±10)  →  PLS to 10 components  →  angles γ·x
      →  10-qubit encoding circuit U(x)  →  one of three readouts:
             ├─ fidelity kernel   k(x,x′) = |⟨ψ(x)|ψ(x′)⟩|²            (one circuit per pair of molecules)
             ├─ projected kernel  RBF kernel on each qubit's ⟨X⟩,⟨Y⟩,⟨Z⟩   (3 circuits per molecule)
             └─ ⟨Z⟩ features       one expectation value per qubit        (1 circuit per molecule)
      →  kernel ridge or ridge regression on √|μ|  →  |μ| in debye
```

| Model | Readout | Role |
|---|---|---|
| **Quantum kernel ridge regression (QKRR), ZZ map** | Fidelity kernel: the overlap of two molecules' states, measured on a device as the probability of reading all zeros after U(x′) then U(x)† | **Headline quantum model** (chosen by CV) |
| QKRR, RY + CZ / RY-RZ | Fidelity kernel | Encoder variants |
| Product-state kernel | Fidelity kernel of the unentangled circuit | Control: is entanglement needed? |
| Projected quantum kernel (Huang et al. 2021) | RBF kernel on each qubit's Bloch vector (⟨X⟩, ⟨Y⟩, ⟨Z⟩) | Cheaper on hardware; resists concentration |
| ⟨Z⟩-feature ridge (the team's original model) | Ridge regression on the 10 values ⟨Zᵢ⟩ | Cheapest on hardware; the model run on IBM hardware |

Kernel ridge regression is used rather than a trained (variational) circuit: its fit is closed-form and convex, so
there is no circuit optimization and no barren plateaus, and it allows a one-to-one comparison with classical RBF
kernel ridge, which differs only in the kernel.

**Exact simulation.** Statevectors of all molecules are computed together by a batched simulator that reads the
Qiskit circuit ([`models/qsim.py`](../src/qm9dipole/models/qsim.py)). It matches Qiskit's own
`qiskit.quantum_info.Statevector` to 10⁻¹² (unit-tested for every gate and encoder) and is about 10× faster, which is
what made cross-validated quantum kernels on 1,000 molecules affordable on a laptop.

**A closed form.** For the two-pass ZZ map, each qubit's ⟨Z⟩ has an exact classical formula
(`quantum_kernel.zz_readout_closed_form`), verified to 3 × 10⁻¹⁴: the ⟨Z⟩-feature model is ridge regression on a fixed
classical feature map. This is a property of shallow circuits with Z readout, not an implementation flaw.

## 7. Fair-comparison protocol

Every quantum kernel is paired with a **classical twin**, RBF kernel ridge, and the two differ only in the kernel:
- **same inputs**: the same Yeo-Johnson → PLS(10) → standardized numbers, fit inside every training fold;
- **same data**: the same nested training sets, seeds and 5-fold CV folds;
- **same tuning**: the same CV metric (pooled out-of-fold MAE in debye), the same refit, and grids of the same size
  (88 candidates each; the quantum angle grid maps onto the RBF bandwidth grid in the small-angle limit);
- **same code**: the quantum tuner reproduces the classical one exactly when given the RBF kernel (unit-tested), so
  a quantum-vs-RBF difference comes from the kernel, not the tuning.

The matched budget applies to the quantum kernels and their RBF twins. The ⟨Z⟩-feature ridge searches 165 settings
(15 ridge penalties × 11 angle scales) against 15 for plain ridge on the same inputs, so that pairing is not an
equal-budget comparison ([notebook 11](../notebooks/11_quantum_feature_ridge.ipynb)).

**Choosing what to report.** The headline quantum model is the quantum configuration with the lowest CV error at
N = 1,000, chosen inside the training data. The reference models are chosen per training size: the best Track B
model by CV, the best Track A model by its `dev` score. Results are mean ± standard deviation over the 3 seeds, plus
paired per-seed differences (quantum − twin), which cancel most of the seed-to-seed variation.

**Metrics.** Mean absolute error (MAE, primary) and root-mean-square error (RMSE), in debye; R² and bias in the
detailed tables. For scale: always predicting the training mean gives MAE ≈ 1.14 D on `test_familiar`.

## 8. From exact simulation to real hardware

Exact statevectors are what a perfect quantum computer would give with infinitely many measurements. Real devices
give neither, so the quantum models were also run under increasingly realistic conditions on identical molecules:
exact → finite shots → simulated device noise → real hardware.

**Finite shots** ([`noise.py`](../src/qm9dipole/noise.py)). Each circuit is run S times and a probability is
estimated as a count over S, with standard error √(p(1 − p)/S). An ideal device's estimate of a kernel entry k is
exactly Binomial(S, k)/S, so shots are drawn from that distribution; 60 real overlap circuits run on Qiskit Aer's
sampler confirm it (z-scores with mean −0.01 and standard deviation 1.00). S ∈ {100, 1,000, 10,000}, 5 repetitions.

**Shot-aware tuning.** A kernel model tuned on exact kernels can choose settings that shot noise destroys (see
[results.md](results.md#5-finite-shots-and-shot-aware-tuning)). The shot-aware variant
(`quantum_kernel.ShotNoisyKernel`) estimates every kernel entry or Bloch coordinate from S shots *inside*
cross-validation, in the final fit and in prediction, so CV picks the settings that work with the noise. The ridge
penalty grid is extended upward (to α = 1,000) for these models, since noise calls for stronger regularization.

**Device noise** ([`noise.py`](../src/qm9dipole/noise.py)). Circuits are transpiled for the device (rewritten into
its native gates, rz, sx, x and the two-qubit cz, between physically connected qubits, at optimization level 2)
and run on Aer with a noise model from a calibration snapshot of an IBM device (`FakeFez`, IBM Heron r2; `FakeQuebec`,
for `ibm_quebec`), using the density-matrix method. Noise pushes every state toward the maximally mixed state, which
shrinks all fidelity-kernel entries by roughly the same factor; one extra circuit per molecule (its overlap with
itself) measures that factor, and a depolarizing correction undoes it.

**Real hardware** ([`hardware.py`](../src/qm9dipole/hardware.py),
[`scripts/hardware_run.py`](../scripts/hardware_run.py)). The ⟨Z⟩-feature ridge ran on IBM's `ibm_quebec` (a
156-qubit Heron processor) through `qiskit-ibm-runtime`'s `SamplerV2` in job mode: the 100 training molecules of
S(0, 100) and 60 test molecules (30 from each quantum test subset), 10 qubits, 1,000 shots each, default options (no
error suppression or mitigation). The ridge readout was then refit on the device's own measurements, on a laptop.
The script plans the job (circuits, shots, estimated QPU time) and submits only with `--submit`, a typed
confirmation and a time cap.

**Cost model** ([`cost.py`](../src/qm9dipole/cost.py)). Circuit depth and two-qubit gate counts come from Qiskit's
transpiler for IBM Heron. QPU time ≈ circuits × shots × (circuit duration + ~2 µs readout + ~250 µs repetition delay),
a lower bound that ignores job and queue overheads. The number of circuits is what separates the models: the
fidelity kernel needs N(N − 1)/2 circuits to train and N per new molecule, the projected kernel 3 per molecule, and
the ⟨Z⟩ ridge 1.

## 9. Evaluation runs

The test sets were scored by [`scripts/final_eval.py`](../scripts/final_eval.py), which retrains every model on the
saved training sets (tuning by CV inside each one) and scores it on the test and development sets. Each run is logged
in [`results/test_runs.csv`](../results/test_runs.csv) with its commit and configuration, and writes
`results/*/test_runNN_*.csv`.

| Run | What it evaluated | Config |
|---|---|---|
| 1 | **Main evaluation**: every model as developed on the development sets, unchanged: the quantum models and their classical comparisons (N ≤ 1,000), Track A to 99,198 labels, finite shots, and FakeFez noise on 60 test molecules (≈ 4.5 h) | [`test_eval.yaml`](../configs/test_eval.yaml) |
| 2 | **Shot-aware quantum kernels**: QKRR at 100 / 1,000 / 10,000 shots and the projected kernel at 100 / 1,000 shots, tuned with the shots inside CV (≈ 20 min) | [`test_eval_shot_aware.yaml`](../configs/test_eval_shot_aware.yaml) |
| 3 | **Hardware rehearsal**: the hardware circuits on the FakeQuebec noise model, locally | `hardware_run.py` |
| 4 | **IBM hardware**: the ⟨Z⟩ ridge on `ibm_quebec` (job `db3da0kvf2bc73csuk60`, 160 circuits × 1,000 shots, 45 s of QPU time) | `hardware_run.py --backend ibm_quebec` |

Run 2 was added after run 1 showed the fidelity-kernel model collapsing under shot noise. Run 1's numbers are
reported unchanged.

**Reproduction checks.** Run 1 also re-scored the development sets: all 464 rows it shares with the development
notebooks match them exactly (difference 0.0 D), so the test numbers belong to the models that were studied. Run 2
reproduces the 180 rows it shares with run 1 exactly.

## 10. Seeds and settings

- **Splits:** split seed 2026. **Training sets:** seeds 0, 1, 2; each fixes the training-set fill order, the 5-fold CV
  folds, the XGBoost search and the network initialization.
- **Simulated shots:** seeded deterministically by (seed, N, shots, repetition), or by seed and inputs for the
  shot-aware kernels. Aer sampler seeds are fixed; transpilation uses `seed_transpiler=0`.
- **Exact simulation:** batched statevectors in float64. **Noise:** Aer `NoiseModel.from_backend`, density-matrix
  method, 1,000 shots. **Hardware:** `ibm_quebec`, `SamplerV2` job mode, 1,000 shots, default options.
- **Provenance:** every results CSV has a `.meta.json` file recording the commit, configuration, seeds and package
  versions it was produced with.

## Glossary

| Term | Meaning |
|---|---|
| **D (debye)** | Unit of dipole moment; 1 D ≈ 0.21 e·Å. Water ≈ 1.85 D; the QM9 median is 2.52 D |
| **MAE / RMSE** | Mean absolute error / root-mean-square error of the predictions, in debye. RMSE weighs large errors more |
| **DFT** | Density functional theory, the quantum-chemistry method that computed QM9's geometries and labels |
| **Z, R** | Atomic numbers and 3D atomic coordinates (Å): the only model inputs |
| **N** | Number of labelled training molecules |
| **CV** | Cross-validation: tuning hyperparameters on held-out folds of the training set only |
| **Familiar / unseen formulas** | Test molecules whose molecular formula does / does not occur in any training set |
| **Track A / Track B** | The best classical models (any size, any input) / classical models restricted to what the quantum models get ([§5](#5-two-classical-references)) |
| **Classical twin** | RBF kernel ridge on exactly the quantum model's inputs, data, folds and tuning budget |
| **PLS** | Partial least squares: a supervised compression to the k input directions most correlated with the target |
| **Kernel ridge regression** | Ridge regression in the feature space defined by a similarity function (kernel) between pairs of molecules |
| **RBF kernel** | The classical Gaussian similarity exp(−γ‖x − x′‖²) |
| **Qubit** | A two-level quantum system; n qubits hold a state of 2ⁿ complex amplitudes |
| **Encoding circuit U(x)** | A fixed quantum circuit whose rotation angles are the inputs, mapping data to a quantum state |
| **Entanglement** | Correlation between qubits that no product of single-qubit states can produce; created here by CX/CZ gates |
| **Fidelity kernel** | The squared overlap \|⟨ψ(x)\|ψ(x′)⟩\|² of two encoded states |
| **QKRR** | Quantum kernel ridge regression: kernel ridge with the fidelity kernel |
| **Projected quantum kernel** | A classical RBF kernel applied to each qubit's measured Bloch vector (⟨X⟩, ⟨Y⟩, ⟨Z⟩) |
| **⟨Z⟩ (expectation value)** | The average of a qubit's ±1 measurement outcomes in the Z basis |
| **Shots** | Repetitions of a circuit; probabilities and expectations are estimated from their outcomes |
| **Transpilation** | Rewriting a circuit into a device's native gates and connectivity |
| **Noise model** | A simulation of a device's gate, decoherence and readout errors from its calibration data |
| **QPU time** | Time charged on the quantum processor; IBM's free Open Plan gives 10 minutes per 28 days |
| **Exponential concentration** | The tendency of fidelity-kernel entries to all approach the same value as qubits are added, which makes molecules indistinguishable |
| **Effective number of parameters (df)** | How much flexibility a fitted model uses; for kernel ridge, the trace of its hat matrix |
