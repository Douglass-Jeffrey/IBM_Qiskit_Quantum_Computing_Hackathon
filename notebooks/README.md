# Notebooks

The notebooks are the project's narrative, numbered in run order. Each opens with the question it answers, what it
reads and writes, and its runtime, and closes with "What we learned". All logic lives in the tested package
`src/qm9dipole/`; the notebooks call it and present the results. Every notebook's executed outputs are committed, so
they can be read on GitHub without running anything.

**Where to start:** [08_quantum_vs_classical](08_quantum_vs_classical.ipynb) is the central experiment, and
[10_test_set_results](10_test_set_results.ipynb) the final scorecard. Notebooks 03–07 build the classical side.

## The pipeline

```
00 setup ─▶ 01 parse + splits ─▶ 02 invariant descriptors
                    │
                    ▼
            03 data analysis ─▶ 04 features ─▶ 05 scaling + compression ─▶ 06 classical models ─▶ 07 diagnostics
                                                                                   │
                                                                                   ▼
                                                             08 quantum vs classical ─▶ 09 shots, noise, cost
                                                                                   │
                     scripts/final_eval.py, scripts/hardware_run.py  (test sets)   ▼
                                                     10 test-set results, 11 the team's ⟨Z⟩ ridge
```

| | Notebook | Question | Runtime |
|---|---|---|---|
| **Data** | [00_setup_and_download](00_setup_and_download.ipynb) | Is the environment right, and are the raw files intact? | ~1 min |
| | [01_parse_and_split](01_parse_and_split.ipynb) | Which molecules are usable, and how do we split them to test familiar and unseen formulas? | ~1 min |
| | [02_invariant_descriptors](02_invariant_descriptors.ipynb) | Can we describe a molecule in a way that ignores rotation, translation and atom order? | ~10 s |
| **Classical** | [03_exploratory_data_analysis](03_exploratory_data_analysis.ipynb) | What in QM9 carries information about the dipole, and what may a model use? | ~4 min |
| | [04_cleaning_and_features](04_cleaning_and_features.ipynb) | Which features, at the molecule and atom level, capture how bond dipoles add up? | ~3 min |
| | [05_scaling_and_compression](05_scaling_and_compression.ipynb) | How to scale features, transform the target, and compress to 10 inputs for 10 qubits? | ~1.5 h |
| | [06_classical_models](06_classical_models.ipynb) | How accurate can a classical model get, and what should a quantum model be compared with? | ~8 h |
| | [07_generalization_and_diagnostics](07_generalization_and_diagnostics.ipynb) | Where do the errors come from, and what did the models learn? | ~10 min |
| **Quantum** | [08_quantum_vs_classical](08_quantum_vs_classical.ipynb) | Does a quantum kernel beat its classical twin on identical inputs? Why or why not? | ~2 h |
| | [09_shots_noise_and_cost](09_shots_noise_and_cost.ipynb) | What would it cost on real hardware, and what do shots and noise do to accuracy? | ~20 min |
| **Test** | [10_test_set_results](10_test_set_results.ipynb) | How do all models do on the held-out test sets, and on IBM hardware? | ~3 min |
| | [11_quantum_feature_ridge](11_quantum_feature_ridge.ipynb) | How does the team's ⟨Z⟩-feature ridge model, the one run on IBM hardware, compare on its own? | seconds |

Runtimes are measured from scratch on an 8-core laptop (October 2026); the long notebooks cache finished fits in `data/processed/cache/` and
resume after an interruption. Notebooks 03–09 use only the training pool and the two development sets. The test sets
are scored by `scripts/final_eval.py` (and `scripts/hardware_run.py`); notebooks 10 and 11 present those results.

**Outputs.** Each notebook writes its tables and figures as `results/<group>/nbNN_*.csv` and
`figures/<group>/nbNN_*.png`, where NN is its number; files from the evaluation script are named `test_runNN_*`. The
index of every output is [results/README.md](../results/README.md).

## Earlier names

The notebooks were renumbered after the hackathon. Git history, commit messages and the run log
(`results/test_runs.csv`) use the earlier names:

| Now | Before | Output prefix before |
|---|---|---|
| 00_setup_and_download | 00_setup_and_data | — |
| 01_parse_and_split | 01_parse_and_splits | `m1_` |
| 02_invariant_descriptors | 02_descriptors_invariance | `m2_` |
| 03_exploratory_data_analysis | explore_01_eda | `explore01_` |
| 04_cleaning_and_features | explore_02_cleaning_and_features | `explore02_` |
| 05_scaling_and_compression | explore_03_standardization_and_dimension | `explore03_` |
| 06_classical_models | explore_04_classical_models | `explore04_` |
| 07_generalization_and_diagnostics | explore_05_generalization_and_diagnostics | `explore05_` |
| 08_quantum_vs_classical | explore_06_quantum_vs_classical | `explore06_` |
| 09_shots_noise_and_cost | explore_07_shots_noise_cost | `explore07_` |
| 10_test_set_results | 07_test_comparison | — |
| 11_quantum_feature_ridge | 07_quantum_feature_ridge (added by yuyuko-s at the deadline) | `qridge_` |
