# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A fork of the [Chemprop](https://github.com/chemprop/chemprop) library — message passing neural networks (MPNNs) for molecular property prediction. Includes both a CLI and a Flask-based web interface for training models on SMILES-based datasets and generating predictions.

## Setup

```bash
conda env create -f environment.yml
conda activate chemprop
pip install -e .
```

## Key Commands

```bash
# Web interface (dev)
chemprop_web          # or: python web.py
# → http://localhost:5000

# Web interface (production)
cd chemprop/web && gunicorn --bind localhost:8000 'wsgi:build_app()'

# Train a model
chemprop_train --data_path data/tox21.csv --dataset_type classification --save_dir tox21_checkpoints

# Predict
chemprop_predict --test_path data/tox21.csv --checkpoint_dir tox21_checkpoints --preds_path preds.csv

# Run all tests
pytest tests/test_integration.py -v

# Run a single test
pytest tests/test_integration.py::ChempropTests::test_train_single_task_regression -v
```

## Architecture

**Entry points** (defined in `setup.py`):
- `chemprop_train` / `chemprop_predict` → `chemprop/train/`
- `chemprop_hyperopt` → `chemprop/hyperparameter_optimization.py`
- `chemprop_interpret` → `chemprop/interpret.py`
- `chemprop_web` → `chemprop/web/run.py`
- `sklearn_train` / `sklearn_predict` → `chemprop/sklearn_train.py`, `chemprop/sklearn_predict.py`

**Training pipeline**: `args.py` (CLI arg parsing via `typed-argument-parser`) → `data/` (SMILES loading, scaffold/random splitting, feature scaling) → `models/model.py` + `models/mpn.py` (MPNN architecture) → `train/run_training.py` (training loop with NoamLR scheduler) → `train/evaluate.py` (task-specific metrics).

**Web interface** (`chemprop/web/`):
- `app/__init__.py` — Flask app factory
- `app/views.py` — all routes; training jobs run in a `multiprocessing` subprocess; progress polled via `/trainingProgress` (JSON endpoint)
- `app/db.py` + `app/schema.sql` — SQLite for user, dataset, checkpoint, and job metadata
- `wsgi.py` — `build_app()` factory used by Gunicorn; accepts `init_db=True` and `demo=True` kwargs
- `config.py` — Flask config constants; GPU detection via `torch.cuda.is_available()`

**Test suite** (`tests/test_integration.py`): one `TestCase` class (`ChempropTests`) with parameterized tests covering regression/classification train+predict, hyperopt, interpret, and the Flask web app. Test data lives in `tests/data/`.
