# Learning molecular dipoles from fewer labels: quantum kernels vs classical models

**Can a quantum machine-learning model predict a molecular property from fewer labelled examples than a classical
one?** We built quantum regression models in [Qiskit](https://www.ibm.com/quantum/qiskit), compared each one with a
classical "twin" that gets exactly the same inputs, data and tuning budget, and ran the most hardware-friendly one on
a 156-qubit IBM quantum processor.

*A team project from Qiskit Fall Fest 2026 at the University of Ottawa (October 3–7, 2026), reorganized afterwards
for readability.*

![Test-set error vs number of training labels](figures/comparison/test_run01_learning_curves.png)

*Prediction error on held-out molecules as the number of training labels grows (log–log; top: mean absolute error,
bottom: root-mean-square error; left: molecular formulas seen in training, right: formulas never seen). Blue: quantum
models on 10 qubits. Orange: their classical twin, RBF kernel ridge on the same 10 inputs; the two lines overlap.
Dashed black: the best classical model at each size. Dotted grey: always predicting the mean.*

## The problem

A molecule's **dipole moment** μ measures how far apart its centres of positive and negative charge sit; it governs
solubility, intermolecular forces and spectroscopy. Its magnitude is measured in **debye (D)**: water has about
1.85 D, and a typical molecule in our data about 2.5 D.

We use **QM9**, 130,683 small organic molecules (up to nine C, N, O or F atoms, plus hydrogens) whose dipoles were
computed with density functional theory (DFT). Every label costs an expensive quantum-chemistry calculation, so the
question is how many labelled molecules a model needs, and whether quantum kernel methods, often proposed for
small-data problems, need fewer. The models see only each molecule's **atomic numbers and 3D coordinates**, and must
give the same answer when a molecule is rotated, moved or has its atoms renumbered, as the true dipole magnitude does.

## Key findings

All numbers are test-set **mean absolute error (MAE) in debye**, averaged over 3 random training sets. For scale,
always predicting the average dipole gives 1.14 D.

1. **The quantum models tie their classical twin.** With 1,000 training labels, quantum kernel ridge regression on
   10 qubits reaches **0.492 D**, and classical kernel ridge with a Gaussian (RBF) kernel on the same 10 inputs
   reaches **0.490 D**. On molecules with familiar formulas, the paired difference is at most 0.005 D at every
   training size. A control circuit with no entanglement does equally well (0.490 D), so the quantum feature map adds
   nothing measurable here.
2. **The best classical model wins as data grows.** A physics-shaped neural network, which predicts a charge on every
   atom and adds up the resulting dipole, reaches **0.040 D** with all 99,198 training labels: 28× better than the
   mean, and far beyond what any quantum model could be trained on with realistic hardware time.
3. **Shot noise breaks the headline quantum model unless it is tuned for it.** A real device estimates every kernel
   entry from a finite number of measurements ("shots"). Tuned on exact simulation, the model's error explodes to
   ≈ 10 D at 1,000 shots per circuit. Tuned with the shot noise inside cross-validation, it scores **0.544 D**.
4. **On IBM hardware, the team's quantum model matched simulation.** On `ibm_quebec` (160 circuits × 1,000 shots,
   45 s of quantum-processor time) it scored **0.778 D** on 60 test molecules, against 0.794 D in exact simulation,
   a difference within sampling error.
5. **Circuit count decides what can run on hardware.** The headline quantum kernel needs one circuit per *pair* of
   molecules: about 1,640 QPU-hours to train on 1,000 molecules and predict the test sets, against 1.7 hours for the
   feature-based model that ran on hardware. IBM's free tier provides 10 minutes per 28 days.

**Why the quantum kernels tie:** cross-validation steers them into a small-angle regime where the quantum kernel
behaves like a smooth classical kernel (same effective number of parameters, 71.4 vs 71.7). At larger angles, the
quantum kernel "concentrates": every molecule starts to look equally dissimilar to every other, more so with more
qubits, and accuracy collapses. Full results and discussion: [docs/results.md](docs/results.md).

## How it works

```
atomic numbers Z + 3D coordinates R
  → 188 rotation- and permutation-invariant features (bond geometry, functional groups, a charge-equilibration
    dipole estimate, radial distributions, Coulomb-matrix eigenvalues)
  → scaling, then partial least squares (PLS) compression to 10 inputs   ← fit on training data only
  → 10-qubit Qiskit encoding circuit U(x): each input becomes a rotation angle
  → one of three quantum readouts:
        fidelity kernel   |⟨ψ(x)|ψ(x′)⟩|²      one circuit per pair of molecules
        projected kernel  each qubit's ⟨X⟩,⟨Y⟩,⟨Z⟩   3 circuits per molecule
        ⟨Z⟩ features      one value per qubit     1 circuit per molecule
  → kernel ridge or ridge regression  →  |μ| in debye
```

- **A fair comparison by construction.** Every quantum kernel is paired with RBF kernel ridge on identical inputs,
  training sets, cross-validation folds and grid size, through the same tuning code (unit-tested to give identical
  results when handed the RBF kernel). A product-state circuit with no entangling gates tests whether entanglement
  matters.
- **Two classical references.** *Track A* is the most accurate classical model we could build, trained on up to all
  99,198 molecules. *Track B* is classical models held to exactly what the quantum models get (≤ 1,000 molecules,
  the same 10 inputs). The quantum models are reported against both.
- **Generalization.** Test molecules come in two kinds: molecules whose formula appears in training, and 93 whole
  formulas (16,818 molecules) that never appear in any training or development data.
- **From simulation to hardware.** Exact statevector simulation (a batched simulator, validated against Qiskit's
  `Statevector` to 10⁻¹²), then finite shots, then IBM device noise models on Qiskit Aer, then the real device.

The details, with a glossary of every term, are in [docs/methods.md](docs/methods.md).

## Results at a glance

Test-set MAE (D), mean ± standard deviation over 3 seeds. The first five rows all use the same 10 inputs; the best
classical model uses any input and any model family.

| Model | N = 100 | N = 300 | N = 1,000 | N = 1,000, unseen formulas |
|---|---|---|---|---|
| **Quantum kernel ridge, ZZ feature map (headline)** | 0.735 ± 0.062 | 0.578 ± 0.011 | **0.492 ± 0.008** | **0.437 ± 0.010** |
| Product-state kernel (no entanglement; control) | 0.732 ± 0.075 | 0.578 ± 0.011 | 0.490 ± 0.009 | 0.433 ± 0.009 |
| Projected quantum kernel | 0.724 ± 0.057 | 0.578 ± 0.013 | 0.493 ± 0.007 | 0.440 ± 0.016 |
| ⟨Z⟩-feature ridge (team model, run on hardware) | 0.731 ± 0.072 | 0.597 ± 0.010 | 0.545 ± 0.011 | 0.493 ± 0.021 |
| **RBF kernel ridge (classical twin)** | 0.730 ± 0.075 | 0.578 ± 0.010 | **0.490 ± 0.009** | **0.433 ± 0.010** |
| Best classical model at that size (Track A) | 0.653 | 0.550 | 0.280 | 0.276 |
| Predicting the mean | 1.156 | 1.142 | 1.138 | 1.254 |

With more data the best classical model keeps improving: 0.095 D at 10,000 labels and 0.040 D at 99,198.

![The team's quantum model, simulated and on IBM hardware](figures/hardware/hardware_comparison.png)

*The same 60 test molecules scored in exact simulation, with simulated shots, under two IBM noise models, and on
the real `ibm_quebec` device (dark bar).*

## Repository guide

The notebooks are the project's narrative: each opens with the question it answers and closes with what we learned.
They are numbered in run order, and each reads the outputs of the previous ones.

| Notebook | What it does |
|---|---|
| [00_setup_and_download](notebooks/00_setup_and_download.ipynb) | Check the environment; download QM9 and verify checksums |
| [01_parse_and_split](notebooks/01_parse_and_split.ipynb) | Parse 133,885 molecules, apply exclusions, build familiar/unseen splits and nested training sets |
| [02_invariant_descriptors](notebooks/02_invariant_descriptors.ipynb) | First descriptors; prove rotation, translation and relabelling invariance (with a negative control) |
| [03_exploratory_data_analysis](notebooks/03_exploratory_data_analysis.ipynb) | Every QM9 field; what an atom's charge depends on; the high-dipole zwitterions |
| [04_cleaning_and_features](notebooks/04_cleaning_and_features.ipynb) | 188 molecule-level and 85 per-atom features, all invariant |
| [05_scaling_and_compression](notebooks/05_scaling_and_compression.ipynb) | Feature scaling, target transform, compression to 10 inputs |
| [06_classical_models](notebooks/06_classical_models.ipynb) | Best classical models up to 99,198 labels; quantum-matched baselines |
| [07_generalization_and_diagnostics](notebooks/07_generalization_and_diagnostics.ipynb) | New formulas, error structure, feature importance, what the charge network learned |
| [08_quantum_vs_classical](notebooks/08_quantum_vs_classical.ipynb) | **The central experiment:** quantum models vs classical twins; qubits, inputs, kernel concentration |
| [09_shots_noise_and_cost](notebooks/09_shots_noise_and_cost.ipynb) | Hardware cost, finite shots and simulated device noise |
| [10_test_set_results](notebooks/10_test_set_results.ipynb) | **Final scorecard:** every model on the test sets, shot-aware tuning, the IBM hardware run |
| [11_quantum_feature_ridge](notebooks/11_quantum_feature_ridge.ipynb) | A focused look at the team's ⟨Z⟩-feature ridge model, the one that ran on IBM hardware |

Notebooks 03–09 use only training and development data; the test sets are scored once per evaluation run by
[`scripts/final_eval.py`](scripts/final_eval.py). More on each notebook: [notebooks/README.md](notebooks/README.md).

```
src/qm9dipole/   all logic, as a tested Python package (notebooks stay thin)
  data.py, splits.py                 download and checksum, parsing, exclusions, familiar/unseen splits
  descriptors.py, features.py, atoms.py   invariant molecule-level and per-atom features
  preprocess.py, evaluate.py         scaling with outlier clipping, cross-validation, learning curves
  models/classical.py, fitters.py    ridge, kernel ridge, random forest, XGBoost behind one fit/predict interface
  models/charge.py                   latent-charge neural network (NumPy, hand-written gradients)
  models/quantum.py                  the team's Qiskit encoding circuits and ⟨Z⟩-feature ridge
  models/qsim.py                     batched exact statevector simulation of Qiskit circuits
  models/quantum_kernel.py           fidelity and projected quantum kernels, shot-aware kernels, tuning
  noise.py, cost.py, hardware.py     shots, Aer noise models, transpiled cost, IBM Runtime jobs
scripts/         final_eval.py (test-set evaluation runs) · hardware_run.py (IBM hardware or noise-model runs)
configs/         splits, notebook settings, and the evaluation runs
results/, figures/   every output, grouped by question (index: results/README.md); each CSV has a provenance file
splits/          molecule IDs of every set and training set
tests/           671 tests, including negative controls for every split and invariance check
docs/            methods, results, reproduction guide
```

## Reproducing

```bash
conda env create -f environment.yml && conda activate qm9dipole    # or any Python 3.12 virtual environment
pip install -r requirements.txt
pytest -q                                                          # 671 tests
```

Then run the notebooks in order, or reproduce the headline comparison in about 45 minutes on a laptop: notebooks 00,
01 and 04, then `python scripts/final_eval.py --sections quantum`. Step-by-step instructions, runtimes and
troubleshooting: [docs/reproducing.md](docs/reproducing.md).

## Team and acknowledgements

- Built by a team at Qiskit Fall Fest 2026 (University of Ottawa), answering the hackathon prompt *"Learn molecular
  dipoles from fewer labels: how much reference data does a model need to predict molecular polarity?"*
  ([prompt slide](docs/prompt_slide.png)).
- Teammate [yuyuko-s](https://github.com/yuyuko-s) wrote the Qiskit encoding circuits (`build_encoding_circuit`) and
  the original quantum-feature ridge model ([`models/quantum.py`](src/qm9dipole/models/quantum.py)), which every
  quantum model here builds on, and its presentation notebook
  ([notebook 11](notebooks/11_quantum_feature_ridge.ipynb)).
- IBM Quantum hardware access (`ibm_quebec`) was provided through PINQ² for the hackathon.
- The state of the repository at the hackathon deadline is tagged `hackathon-submission`.

## References

**Data.** R. Ramakrishnan, P. O. Dral, M. Rupp, O. A. von Lilienfeld, *Quantum chemistry structures and properties
of 134 kilo molecules*, Sci. Data 1, 140022 (2014), figshare collection 978904. L. Ruddigkeit, R. van Deursen,
L. C. Blum, J.-L. Reymond, *Enumeration of 166 billion organic small molecules in the chemical universe database
GDB-17*, J. Chem. Inf. Model. 52, 2864 (2012).

**Quantum machine learning.**
- V. Havlíček et al., *Supervised learning with quantum-enhanced feature spaces*, Nature 567, 209 (2019) (ZZ feature
  map, quantum kernels).
- M. Schuld, N. Killoran, *Quantum machine learning in feature Hilbert spaces*, Phys. Rev. Lett. 122, 040504 (2019).
- H.-Y. Huang et al., *Power of data in quantum machine learning*, Nat. Commun. 12, 2631 (2021) (projected quantum
  kernel).
- S. Thanasilp, S. Wang, M. Cerezo, Z. Holmes, *Exponential concentration in quantum kernel methods*, Nat. Commun.
  15, 5200 (2024).
- A. Javadi-Abhari et al., *Quantum computing with Qiskit*, arXiv:2405.08810 (2024).

**Chemistry and machine learning.**
- M. Rupp, A. Tkatchenko, K.-R. Müller, O. A. von Lilienfeld, Phys. Rev. Lett. 108, 058301 (2012) (Coulomb matrix).
- A. K. Rappé, W. A. Goddard III, *Charge equilibration for molecular dynamics simulations*, J. Phys. Chem. 95, 3358
  (1991) (QEq).
- O. T. Unke, M. Meuwly, *PhysNet*, J. Chem. Theory Comput. 15, 3678 (2019); M. Gastegger, J. Behler,
  P. Marquetand, Chem. Sci. 8, 6924 (2017) (latent-charge dipole models; ours is an independent implementation).
- C. Cortes, M. Mohri, A. Rostamizadeh, *Algorithms for learning kernels based on centered alignment*, JMLR 13, 795
  (2012) (kernel–target alignment).
- I.-K. Yeo, R. A. Johnson, Biometrika 87, 954 (2000) (Yeo-Johnson transform); S. Wold, M. Sjöström, L. Eriksson,
  *PLS-regression: a basic tool of chemometrics*, Chemom. Intell. Lab. Syst. 58, 109 (2001); T. Chen, C. Guestrin,
  *XGBoost: a scalable tree boosting system*, KDD (2016); F. Pedregosa et al., *Scikit-learn*, JMLR 12, 2825 (2011).
