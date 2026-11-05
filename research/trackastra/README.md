# Trackastra research: fine-tuning and tracking decoders

This directory records completed Windows experiments on Fluo-N2DH-GOWT1. It is separate from the team's `src/` baseline and Cellpose modules. It contains no pretrained or fine-tuned weights, CTC images, datasets, caches, or virtual environments.

**All scores here use sequence `02` as a validation sequence with `02_GT/TRA` masks. They are not independent test-set results.** The GT/TRA masks are oracle detections. They isolate association quality and are not comparable with the team's Cellpose-mask operating point without clearly labeling the different input masks.

## Results

- [Fine-tuning and eight-run grid](reports/grid_search.md): six CT/TRA ties; the selected checkpoint is `e10_s89_lr1e-05_172ea7de19d3`, chosen by Validation Loss among the tied runs.
- [Three-seed stability and cache reproducibility](reports/reproducibility.md): seeds 43, 44, 20839; CT `0.893728 ± 0.006007` and TRA `0.996612 ± 0.000020` (sample SD, n=3). The manual seed-20839 run was reproduced exactly when its cache was reused; the root cache did not reproduce it.
- [Greedy versus ILP](reports/greedy_vs_ilp.md): `general_2d + ILP` has the highest CT (`0.964912`) and TRA (`0.999359`) on this one validation sequence. ILP keeps eight detections omitted by Greedy, but selects more wrong edges.
- [Four-row comparison CSV](results/greedy_vs_ilp.csv), [changed edges](results/changed_edges.csv), [solver timing](results/solver_benchmark.json), and [curated result directory](results/).

## Layout

| Path | Purpose |
|---|---|
| `configs/finetune_config.example.yaml` | Local-path template retaining the original model, training and validation settings |
| `scripts/run_finetune_experiments.py` | Grid and seed runner; starts every attempt from `general_2d`, records best checkpoint and CTC metrics |
| `scripts/analyze_tracking_graphs.py` | Compare six tied graphs, three seed reruns and the manual run |
| `scripts/compare_greedy_ilp.py` | Four model × decoder runs, independent CTC directories, BC/CT/TRA/AOGM and graph errors |
| `scripts/curate_results.py` | Export path-free, small CSV/JSON/graph artifacts from local completed runs |
| `results/` | Numerical results, checkpoint hashes, changed/error edge tables and raw metric CSVs; no full GT graph or TIFF masks |
| `reports/` | Findings and caveats |

## Reproduce the workflow

Use a separate Python environment with the **official Trackastra 0.5.6 checkout** (`aa57a95` for the recorded run), its training dependencies, TensorBoard, PyYAML and py-ctcmetrics 1.3.3. ILP needs Motile and a working ilpy solver backend; the recorded environment used Motile 1.0.1, ilpy 0.6.0, and had gurobipy 13.0.3 installed. Check the solver license/backend locally before rerunning. The target repository's `requirements.txt` remains dedicated to the existing scaffold.

Copy `configs/finetune_config.example.yaml` to a local, untracked YAML and set the three paths (`model`, `input_train`, `input_val`). Download CTC data separately and keep it under ignored `data/`. The original checkpoint is **not** in this repository; place `general_2d` in an external model directory. From this repository root, Windows PowerShell examples are:

```powershell
$py = "C:\path\to\trackastra-env\python.exe"
$official = "C:\path\to\official-trackastra"
$config = "C:\path\to\local-finetune-config.yaml"

# Review the original eight combinations without training.
& $py research/trackastra/scripts/run_finetune_experiments.py --config $config --trackastra-root $official --output-root runs/trackastra/finetune_sweep_01 --epochs 2 10 --train-samples 16 89 --lrs 1e-5 5e-6 --seed 42 --dry-run

# Only if intentionally reproducing the historical training: remove --dry-run.
# Then run three fixed-parameter seed repetitions into a separate output root.
& $py research/trackastra/scripts/run_finetune_experiments.py --config $config --trackastra-root $official --output-root runs/trackastra/finetune_seed_repeats --epochs 10 --train-samples 89 --lrs 1e-5 --seed 43

# Evaluate the already selected best model under both decoders; no training.
& $py research/trackastra/scripts/compare_greedy_ilp.py --pretrained C:\path\to\general_2d --finetuned C:\path\to\selected\train --sequence data/Fluo-N2DH-GOWT1/02 --gt data/Fluo-N2DH-GOWT1/02_GT --output-root runs/trackastra/greedy_vs_ilp_02 --device cuda --benchmark-solvers
```

The runner's `--cache-dir` pins the official training cache (default `<trackastra-root>/.cache`). Cache state affects crop randomness. Repeating a seed with a different cache does not guarantee the same weights. Training uses official `scripts/train.py`; best-epoch `model.pt` is used for inference, while `last.ckpt` is only a resume checkpoint. `ctc_evaluate` must be on PATH or supplied with `--ctc-evaluate <absolute executable path>`. New runs are written under ignored `runs/`; the fine-tuning runner skips completed runs with a valid checkpoint and CTC result, while the decoder comparison also checks checkpoint hashes before skipping. Both keep unrelated CTC outputs intact.

To regenerate the small committed tables from a local copy of the completed experiments, use `scripts/curate_results.py --local-scripts <official-checkout>/scripts`. The local run directory must contain the original output folders; this command only reads them and writes metadata. The committed reports and numeric results remain useful when the large private artifacts are unavailable.

## Later Cellpose integration

The existing [`src/run_trackastra.py`](../../src/run_trackastra.py) already accepts `--img` and `--masks` and writes a CTC `<seq>_RES` directory for pretrained models. A future fine-tuned integration should accept a **model folder containing `model.pt`, `config.yaml`, and `train_config.yaml`**, plus `--img`, `--masks`, `--mode`, `--device`, and `--out`. `--masks` should be a sequence of 0-based, per-frame integer instance-label TIFFs, one per raw image, with zero as background and matching spatial shape. Track IDs need not be consistent across input frames. Keep Greedy and ILP outputs in distinct directories, then score both with the same `src.evaluate`/py-ctcmetrics path and report the mask source (`GT/TRA` versus Cellpose) beside each metric. Do not use GT/TRA masks as Cellpose outputs or compare the two conditions as if they had identical detections.
