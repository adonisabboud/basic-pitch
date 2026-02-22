# Basic Pitch Repository Overview

This document is a practical walkthrough of the repository—from first contact to a full-system mental model.

## 1) What this repository is

`basic-pitch` is a Python library and CLI for automatic music transcription (audio → note events/MIDI), designed to run a compact model in multiple runtimes (TensorFlow, CoreML, TFLite, ONNX).

At a high level:

1. Audio file is loaded and chunked into fixed windows.
2. A model predicts three time-frequency outputs: `note`, `onset`, and `contour`.
3. Post-processing converts these activations into note events.
4. Results are exported as MIDI, optional CSV/NPZ, and optional sonified WAV.

## 2) Fast orientation: important top-level files

- `README.md`: user-facing install/usage overview.
- `pyproject.toml`: package metadata, dependencies, and CLI entry points.
- `basic_pitch/`: core library code.
- `tests/`: unit/integration tests and fixture data.
- `basic_pitch/saved_models/icassp_2022/`: bundled model artifacts in multiple serialization formats.

## 3) Package map (what each module does)

### Inference path (most users)

- `basic_pitch/predict.py`
  - CLI argument parsing (`basic-pitch ...`).
  - Selects model serialization and calls inference orchestration.

- `basic_pitch/inference.py`
  - Runtime abstraction class `Model` for TF/CoreML/TFLite/ONNX.
  - Audio windowing and overlap logic.
  - Prediction and file writing helpers.

- `basic_pitch/note_creation.py`
  - Converts model activation outputs into MIDI notes/events.
  - Applies thresholds, minimum note length, pitch bend handling.

- `basic_pitch/constants.py`
  - Shared constants (sample rates, frame geometry, etc.).

### Training path (contributors/researchers)

- `basic_pitch/train.py`
  - Training loop and callbacks.
  - Dataset sampling / TensorBoard / checkpoints.

- `basic_pitch/models.py`
  - Keras model architecture, losses, and feature extraction layers.

- `basic_pitch/nn.py`
  - Reusable neural layers (e.g., harmonic stacking).

- `basic_pitch/layers/*`
  - Low-level signal/math ops used by model definition.

### Data prep path

- `basic_pitch/data/README.md`
  - Notes on dataset preparation and Beam pipeline usage.

- `basic_pitch/data/pipeline.py`, `tf_example_serialization.py`, `tf_example_deserialization.py`
  - Serialization/deserialization and dataset construction.

- `basic_pitch/data/datasets/*.py`
  - Dataset-specific ingestion/adapters.

## 4) End-to-end execution flow

### CLI flow

`basic_pitch/predict.py`:

1. Parse args (paths + thresholds + output options).
2. Resolve model path:
   - default serialized model path from `basic_pitch.__init__`
   - or explicit `--model-path`
   - or explicit `--model-serialization`.
3. Build `Model(...)` from `basic_pitch/inference.py`.
4. Call `predict_and_save(...)`.

### Inference flow internals

`basic_pitch/inference.py` does the heavy lifting:

1. `get_audio_input(...)` loads mono audio at the internal sample rate and adds overlap padding.
2. `window_audio_file(...)` yields fixed-size windows.
3. `Model.predict(...)` runs forward pass on each window.
4. `unwrap_output(...)` removes overlap frames and stitches windows back into a full sequence.
5. `note_creation.model_output_to_notes(...)` transforms activations into note events + MIDI.
6. Optional exports are written (MIDI, note events CSV, model outputs NPZ, sonified WAV).

## 5) Runtime selection strategy

The package supports multiple runtimes to keep installation light and cross-platform:

- TensorFlow
- CoreML
- TensorFlow Lite
- ONNX Runtime

The default model file (`ICASSP_2022_MODEL_PATH`) is chosen by availability in that order. This behavior is defined in `basic_pitch/__init__.py`, while actual loading logic and fallback error messaging live in `basic_pitch/inference.py`.

## 6) Testing strategy in this repository

Tests are split by concern:

- Core behavior tests:
  - `tests/test_inference.py`
  - `tests/test_note_creation.py`
  - `tests/test_nn.py`
  - `tests/test_callbacks.py`

- Data pipeline tests under `tests/data/`:
  - serialization/deserialization
  - dataset adapters
  - pipeline mechanics

- `tests/resources/` provides small fixture files (dummy metadata, short audio snippets, expected outputs).

## 7) Extension points (where to change things)

- Change CLI behavior/options: `basic_pitch/predict.py`
- Change model runtime loading behavior: `basic_pitch/inference.py` (`Model` class)
- Tune note extraction or pitch bend logic: `basic_pitch/note_creation.py`
- Modify model architecture/losses: `basic_pitch/models.py` + `basic_pitch/nn.py`
- Add/support datasets: `basic_pitch/data/datasets/` + pipeline serialization code

## 8) Contributor mental model

Think of the repository as **three mostly independent layers**:

1. **Model + DSP layer** (`models.py`, `nn.py`, `layers/`, `constants.py`)
2. **Inference/post-processing layer** (`inference.py`, `note_creation.py`)
3. **Interface/orchestration layer** (CLI in `predict.py`, packaging in `pyproject.toml`, tests)

This separation is why the project can ship one algorithmic core while supporting multiple runtimes and both library and CLI workflows.

## 9) Suggested first reading order for a new engineer

1. `README.md` (user perspective)
2. `basic_pitch/predict.py` (entrypoint)
3. `basic_pitch/inference.py` (execution pipeline)
4. `basic_pitch/note_creation.py` (post-processing rules)
5. `basic_pitch/models.py` and `basic_pitch/nn.py` (model internals)
6. `tests/test_inference.py` + `tests/test_note_creation.py` (behavior expectations)
7. `basic_pitch/data/README.md` (training/data pipeline context)

