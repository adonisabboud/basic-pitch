# Basic Pitch Repository Overview

A practical guide to understand this repository from first contact to full architecture.

## What this project is

`basic-pitch` is a Python library and CLI for automatic music transcription (audio to MIDI/note events). It ships a lightweight model and supports multiple runtimes (TensorFlow, CoreML, TFLite, ONNX).

Core flow:
1. Load audio.
2. Window audio into fixed-size chunks.
3. Run model inference (`note`, `onset`, `contour` outputs).
4. Post-process activations into note events.
5. Save MIDI and optional extra artifacts (CSV/NPZ/WAV sonification).

## Top-level orientation

- `README.md`: install + usage.
- `pyproject.toml`: packaging metadata, deps, and CLI entrypoints.
- `basic_pitch/`: source code.
- `tests/`: behavior and data pipeline tests.
- `basic_pitch/saved_models/icassp_2022/`: bundled model files.

## Code map by responsibility

### Inference path

- `basic_pitch/predict.py`
  - CLI entrypoint (`basic-pitch`).
  - Parses arguments and selects model serialization/path.

- `basic_pitch/inference.py`
  - Runtime loader abstraction (`Model`) for TF/CoreML/TFLite/ONNX.
  - Audio chunking/overlap handling.
  - Prediction orchestration and output writing.

- `basic_pitch/note_creation.py`
  - Converts raw model activations into note events and MIDI.
  - Applies thresholds, note length filtering, pitch-bend logic.

- `basic_pitch/constants.py`
  - Shared constants used across inference/model code.

### Training path

- `basic_pitch/train.py`
  - Training loop setup, callbacks, logging, checkpoints.

- `basic_pitch/models.py`
  - Keras model architecture and losses.

- `basic_pitch/nn.py`
  - Reusable NN layers (e.g., harmonic stacking).

- `basic_pitch/layers/*`
  - Lower-level signal/math layers used in model construction.

### Data preparation path

- `basic_pitch/data/README.md`
  - Data/training notes and Beam pipeline options.

- `basic_pitch/data/pipeline.py`
- `basic_pitch/data/tf_example_serialization.py`
- `basic_pitch/data/tf_example_deserialization.py`
  - Dataset pipeline + TF example read/write logic.

- `basic_pitch/data/datasets/*.py`
  - Dataset-specific adapters/download-processing helpers.

## End-to-end runtime flow

### CLI

`basic_pitch/predict.py` does:
1. Parse CLI args.
2. Validate input/output paths.
3. Resolve model path/serialization.
4. Build `Model(...)`.
5. Call `predict_and_save(...)`.

### Inference internals

`basic_pitch/inference.py` pipeline:
1. `get_audio_input(...)`: load mono audio at internal sample rate and apply overlap padding.
2. `window_audio_file(...)`: generate fixed windows.
3. `Model.predict(...)`: run inference per window.
4. `unwrap_output(...)`: remove overlap and stitch frame outputs.
5. `note_creation.model_output_to_notes(...)`: convert activations to notes/MIDI.
6. Save requested artifacts.

## Runtime selection behavior

Default model/runtime preference is availability-based in this order:
1. TensorFlow
2. CoreML
3. TensorFlow Lite
4. ONNX

Selection defaults are wired in `basic_pitch/__init__.py`; loading/fallback behavior lives in `basic_pitch/inference.py`.

## Testing layout

Core tests:
- `tests/test_inference.py`
- `tests/test_note_creation.py`
- `tests/test_nn.py`
- `tests/test_callbacks.py`

Data tests:
- `tests/data/*.py`

Fixtures:
- `tests/resources/*`

## Where to modify behavior

- CLI arguments and command UX: `basic_pitch/predict.py`
- Runtime/model loading: `basic_pitch/inference.py`
- Note extraction heuristics: `basic_pitch/note_creation.py`
- Model architecture/loss: `basic_pitch/models.py`, `basic_pitch/nn.py`
- Data ingestion support: `basic_pitch/data/datasets/*` and data pipeline modules

## Suggested reading order for onboarding

1. `README.md`
2. `basic_pitch/predict.py`
3. `basic_pitch/inference.py`
4. `basic_pitch/note_creation.py`
5. `basic_pitch/models.py` + `basic_pitch/nn.py`
6. `tests/test_inference.py` + `tests/test_note_creation.py`
7. `basic_pitch/data/README.md`
