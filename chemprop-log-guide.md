# Chemprop Training Log Guide

## 1. Running Chemprop with a Save Directory

The `--save_dir` flag controls where all training outputs land, including the structured training log.

**Regression example:**
```bash
chemprop_train \
  --data_path data/delaney.csv \
  --dataset_type regression \
  --save_dir output/delaney_run
```

**Classification example:**
```bash
chemprop_train \
  --data_path data/tox21.csv \
  --dataset_type classification \
  --save_dir output/tox21_run
```

**With cross-validation (multiple folds):**
```bash
chemprop_train \
  --data_path data/tox21.csv \
  --dataset_type classification \
  --num_folds 5 \
  --save_dir output/tox21_cv
```

After training, the save directory contains:

```
output/tox21_run/
├── train.log           ← structured JSONL training log (covered below)
├── args.json           ← full argument snapshot
├── test_scores.json    ← final test-set scores
├── test_preds.csv      ← per-molecule predictions (if --save_preds)
└── fold_0/
    └── model_0/
        └── model.pt    ← saved checkpoint
```

---

## 2. The `train.log` Format

`train.log` is a **JSONL file** (one JSON object per line). Every line has a `t` field with a UTC timestamp. Lines are appended in real time as training progresses, so you can `tail -f train.log` to watch a live run.

There are four event types, emitted in this order:

### `run_start`
Emitted once per fold, before the first epoch begins.

| Field | Type | Description |
|---|---|---|
| `event` | string | `"run_start"` |
| `t` | string | UTC timestamp (`YYYY-MM-DDTHH:MM:SSZ`) |
| `fold` | int | Fold index (0-based) |
| `num_folds` | int | Total number of folds |
| `total_epochs` | int | Number of training epochs (`--epochs`) |
| `steps_per_epoch` | int | Number of gradient steps per epoch |
| `train_rows` | int | Molecules in the training split |
| `val_rows` | int | Molecules in the validation split |
| `test_rows` | int | Molecules in the test split |
| `batch_size` | int | Batch size (`--batch_size`) |
| `lr` | float | Peak learning rate (`--max_lr`) |

### `step`
Emitted every `--log_frequency` gradient steps (default: every 10 steps) within an epoch.

| Field | Type | Description |
|---|---|---|
| `event` | string | `"step"` |
| `t` | string | UTC timestamp |
| `fold` | int | Fold index |
| `epoch` | int | Current epoch (1-based) |
| `step` | int | Step within this epoch (1-based) |
| `global_step` | int | Total gradient steps since training started |
| `train_loss` | float | Average training loss over the last `--log_frequency` steps |

### `epoch_end`
Emitted once at the end of every epoch, after validation scoring.

| Field | Type | Description |
|---|---|---|
| `event` | string | `"epoch_end"` |
| `t` | string | UTC timestamp |
| `fold` | int | Fold index |
| `epoch` | int | Epoch just completed (1-based) |
| `global_step` | int | Total gradient steps so far |
| `train_loss` | float | Average training loss for the full epoch |
| `val_loss` | float | Validation metric score (e.g. RMSE, AUC) |
| `is_best` | bool | Whether this epoch achieved the best validation score so far |
| `elapsed_s` | int | Seconds elapsed since `run_start` |

### `checkpoint`
Emitted immediately after `epoch_end` when a new best validation score is reached.

| Field | Type | Description |
|---|---|---|
| `event` | string | `"checkpoint"` |
| `t` | string | UTC timestamp |
| `fold` | int | Fold index |
| `epoch` | int | Epoch at which the checkpoint was saved |
| `path` | string | Absolute path to the saved `model.pt` file |

### `run_end`
Emitted once per fold after all epochs complete and the test set is evaluated.

| Field | Type | Description |
|---|---|---|
| `event` | string | `"run_end"` |
| `t` | string | UTC timestamp |
| `fold` | int | Fold index |
| `best_epoch` | int | Epoch with the best validation score (1-based) |
| `best_val_loss` | float | Best validation metric achieved |
| `total_steps` | int | Total gradient steps across all epochs |
| `elapsed_s` | int | Total seconds for this fold |

---

## 3. Example `train.log`

A short regression run (`--epochs 3`, single fold, ~1100 training molecules):

```jsonl
{"event": "run_start", "fold": 0, "num_folds": 1, "total_epochs": 3, "steps_per_epoch": 87, "train_rows": 1027, "val_rows": 128, "test_rows": 257, "batch_size": 50, "lr": 0.001, "t": "2026-05-05T14:00:01Z"}
{"event": "step", "fold": 0, "epoch": 1, "step": 10, "global_step": 10, "train_loss": 1.183042, "t": "2026-05-05T14:00:04Z"}
{"event": "step", "fold": 0, "epoch": 1, "step": 20, "global_step": 20, "train_loss": 0.974561, "t": "2026-05-05T14:00:07Z"}
{"event": "step", "fold": 0, "epoch": 1, "step": 30, "global_step": 30, "train_loss": 0.821403, "t": "2026-05-05T14:00:10Z"}
{"event": "step", "fold": 0, "epoch": 1, "step": 40, "global_step": 40, "train_loss": 0.743218, "t": "2026-05-05T14:00:13Z"}
{"event": "step", "fold": 0, "epoch": 1, "step": 50, "global_step": 50, "train_loss": 0.698774, "t": "2026-05-05T14:00:16Z"}
{"event": "step", "fold": 0, "epoch": 1, "step": 60, "global_step": 60, "train_loss": 0.662105, "t": "2026-05-05T14:00:19Z"}
{"event": "step", "fold": 0, "epoch": 1, "step": 70, "global_step": 70, "train_loss": 0.631890, "t": "2026-05-05T14:00:22Z"}
{"event": "step", "fold": 0, "epoch": 1, "step": 80, "global_step": 80, "train_loss": 0.607341, "t": "2026-05-05T14:00:25Z"}
{"event": "epoch_end", "fold": 0, "epoch": 1, "global_step": 87, "train_loss": 0.798312, "val_loss": 1.124587, "is_best": true, "elapsed_s": 26, "t": "2026-05-05T14:00:27Z"}
{"event": "checkpoint", "fold": 0, "epoch": 1, "path": "/home/user/output/delaney_run/fold_0/model_0/model.pt", "t": "2026-05-05T14:00:27Z"}
{"event": "step", "fold": 0, "epoch": 2, "step": 10, "global_step": 97, "train_loss": 0.581204, "t": "2026-05-05T14:00:30Z"}
{"event": "step", "fold": 0, "epoch": 2, "step": 20, "global_step": 107, "train_loss": 0.554897, "t": "2026-05-05T14:00:33Z"}
{"event": "step", "fold": 0, "epoch": 2, "step": 30, "global_step": 117, "train_loss": 0.531042, "t": "2026-05-05T14:00:36Z"}
{"event": "step", "fold": 0, "epoch": 2, "step": 40, "global_step": 127, "train_loss": 0.512678, "t": "2026-05-05T14:00:39Z"}
{"event": "step", "fold": 0, "epoch": 2, "step": 50, "global_step": 137, "train_loss": 0.498123, "t": "2026-05-05T14:00:42Z"}
{"event": "step", "fold": 0, "epoch": 2, "step": 60, "global_step": 147, "train_loss": 0.481956, "t": "2026-05-05T14:00:45Z"}
{"event": "step", "fold": 0, "epoch": 2, "step": 70, "global_step": 157, "train_loss": 0.470318, "t": "2026-05-05T14:00:48Z"}
{"event": "step", "fold": 0, "epoch": 2, "step": 80, "global_step": 167, "train_loss": 0.461204, "t": "2026-05-05T14:00:51Z"}
{"event": "epoch_end", "fold": 0, "epoch": 2, "global_step": 174, "train_loss": 0.519847, "val_loss": 0.873241, "is_best": true, "elapsed_s": 53, "t": "2026-05-05T14:00:54Z"}
{"event": "checkpoint", "fold": 0, "epoch": 2, "path": "/home/user/output/delaney_run/fold_0/model_0/model.pt", "t": "2026-05-05T14:00:54Z"}
{"event": "step", "fold": 0, "epoch": 3, "step": 10, "global_step": 184, "train_loss": 0.443871, "t": "2026-05-05T14:00:57Z"}
{"event": "step", "fold": 0, "epoch": 3, "step": 20, "global_step": 194, "train_loss": 0.431056, "t": "2026-05-05T14:01:00Z"}
{"event": "step", "fold": 0, "epoch": 3, "step": 30, "global_step": 204, "train_loss": 0.421893, "t": "2026-05-05T14:01:03Z"}
{"event": "step", "fold": 0, "epoch": 3, "step": 40, "global_step": 214, "train_loss": 0.413047, "t": "2026-05-05T14:01:06Z"}
{"event": "step", "fold": 0, "epoch": 3, "step": 50, "global_step": 224, "train_loss": 0.404982, "t": "2026-05-05T14:01:09Z"}
{"event": "step", "fold": 0, "epoch": 3, "step": 60, "global_step": 234, "train_loss": 0.397841, "t": "2026-05-05T14:01:12Z"}
{"event": "step", "fold": 0, "epoch": 3, "step": 70, "global_step": 244, "train_loss": 0.391204, "t": "2026-05-05T14:01:15Z"}
{"event": "step", "fold": 0, "epoch": 3, "step": 80, "global_step": 254, "train_loss": 0.385671, "t": "2026-05-05T14:01:18Z"}
{"event": "epoch_end", "fold": 0, "epoch": 3, "global_step": 261, "train_loss": 0.412038, "val_loss": 0.881043, "is_best": false, "elapsed_s": 80, "t": "2026-05-05T14:01:21Z"}
{"event": "run_end", "fold": 0, "best_epoch": 2, "best_val_loss": 0.873241, "total_steps": 261, "elapsed_s": 81, "t": "2026-05-05T14:01:21Z"}
```

### Parsing the log

```python
import json

with open("output/delaney_run/train.log") as f:
    events = [json.loads(line) for line in f]

epoch_ends = [e for e in events if e["event"] == "epoch_end"]
for e in epoch_ends:
    print(f"Epoch {e['epoch']:3d}  train={e['train_loss']:.4f}  val={e['val_loss']:.4f}  best={e['is_best']}")
```

Output:
```
Epoch   1  train=0.7983  val=1.1246  best=True
Epoch   2  train=0.5198  val=0.8732  best=True
Epoch   3  train=0.4120  val=0.8810  best=False
```

### Watching a live run

```bash
tail -f output/delaney_run/train.log | python -c "
import sys, json
for line in sys.stdin:
    e = json.loads(line)
    if e['event'] == 'epoch_end':
        print(f\"Epoch {e['epoch']} | val={e['val_loss']:.4f} | best={e['is_best']}\")
"
```
