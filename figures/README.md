# Figures

Grouped like [../results/](../results/README.md). `nbNN_*` figures are made by notebook NN from development data;
`test_run*` figures and `hardware_comparison.png` by [notebook 10](../notebooks/10_test_set_results.ipynb) from the
test sets.

| Folder | Key figures |
|---|---|
| [comparison/](comparison/): quantum vs same-size classical | `test_run01_learning_curves.png` (**headline**: test error vs training labels), `test_run02_shot_aware.png` (shot-aware quantum models), `nb08_learning_curves.png`, `nb08_qubits.png` (error vs qubits), `nb08_extended.png` (to N = 10,000), `nb08_concentration.png` (kernel concentration), `nb08_spectra.png`, `nb08_models_at_nmax.png` |
| [hardware/](hardware/): shots, noise, cost, IBM hardware | `hardware_comparison.png` (**exact vs simulated noise vs `ibm_quebec`**), `test_run01_shots.png` (error vs shots), `nb09_qpu_time.png`, `nb09_shots.png`, `nb09_noise.png` |
| [classical/](classical/): most accurate classical models | `test_run01_track_a.png` (test learning curves to 99,198 labels), `nb06_learning_curves.png`, `nb07_*.png` (parity, learned charges, new-formula gap, feature importance) |
| [development/](development/): data and features | `nb01_pool_overview.png`, `nb03_*.png` (data analysis), `nb04_engineered_information.png`, `nb05_*.png` (scaling, compression) |
