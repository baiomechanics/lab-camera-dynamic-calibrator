# Repository Structure

## Directory Tree

```
lab-camera-dynamic-calibrator/
├── README.md                     # Main documentation (31.5 KB)
├── HOWTO.md                      # Quick usage guide
├── LICENSE                       # MIT License
├── argument.py                   # Shared CLI argument parser
├── conda_linux.yaml              # Conda environment definition
│
├── core/                         # Shared library modules
│   ├── __init__.py               # Re-exports all submodule APIs
│   ├── skeletons.py              # Joint/bone definitions (3 formats)
│   ├── geometry.py               # Triangulation, projection, R/T math
│   ├── poses_io.py               # JSON I/O for poses and cameras
│   ├── filtering.py              # Visibility/orientation helpers
│   └── gpu.py                    # GPU auto-selection utility
│
├── pose/                         # Pose estimation engines
│   ├── metrabs_inference.py      # MeTRAbs 87-joint extraction (recommended)
│   ├── rtmlib_inference.py       # RTMPose 2D detection (legacy)
│   └── inference.py              # VideoPose3D 3D lifting (legacy)
│
├── calibration/                  # Extrinsic calibration algorithms
│   ├── calib_linear.py           # Linear init (Procrustes / collinearity)
│   └── ba.py                     # Bundle Adjustment optimizer
│
├── postprocessing/               # Evaluation, scaling, visualization
│   ├── evaluate_calibration.py   # MRE computation, TOML export
│   ├── scale_scene.py            # Metric scaling, gravity alignment
│   └── visualize_results.py      # 3D GIF/MP4 rendering
│
├── scripts/                      # Pipeline orchestration
│   ├── calibrate.sh              # Main entry point (bash orchestrator)
│   ├── run_calib_linear.py       # Chunked linear calibration
│   ├── detect_outlier_frames.py  # Auto outlier detection
│   ├── run_ba.py                 # BA runner with OOM auto-retry
│   └── setup_models.sh           # VideoPose3D download script
│
├── tools/                        # Utilities
│   └── create_cameras_from_toml.py  # TOML -> JSON camera converter
│
├── utils/                        # Helper scripts
│   ├── convert_calib_rotation.py # Rotate intrinsics 90 degrees
│   └── rotate_video.py           # Rotate video frames
│
├── config/
│   └── config.yaml               # Auto-generated session configuration
│
├── demo/                         # Demo dataset (4 cameras, 100 frames)
│   ├── cam01.mp4 ... cam04.mp4
│   └── Calib_scene.toml          # Demo intrinsics
│
├── input/                        # User input directory (git-ignored)
├── output/                       # Calibration results (git-ignored)
│
├── legacy/archive/               # Archived legacy scripts (27 files)
│
├── img/                          # Documentation images
│
└── third_party/                  # Git submodules (VideoPose3D)
```

## Module Responsibilities

### Core Library (`core/`)

| Module | Lines | Purpose |
|--------|-------|---------|
| `skeletons.py` | 390 | Joint index dictionaries (OP-25, MeTRAbs-26, bml_movi_87), bone connectivity, format auto-detection |
| `geometry.py` | 177 | DLT triangulation, 3D-to-2D projection, R/T inversion, cheirality tests |
| `poses_io.py` | 165 | JSON load/save for poses (2D/3D), camera K/R/t, backward-compatible skeleton loading |
| `filtering.py` | 66 | Per-frame joint visibility masks, bone orientation extraction |
| `gpu.py` | 47 | GPU auto-selection (scans free memory, sets CUDA_VISIBLE_DEVICES) |

### Pose Estimation (`pose/`)

| Module | Lines | Purpose |
|--------|-------|---------|
| `metrabs_inference.py` | 548 | MeTRAbs extraction: bml_movi_87 -> 2D+3D metric, quality filters, smoothing |
| `rtmlib_inference.py` | 350 | RTMPose (Halpe26) 2D keypoint extraction via ONNX |
| `inference.py` | 322 | VideoPose3D: 2D-to-3D temporal lifting (only used with RTMPose) |

### Calibration (`calibration/`)

| Module | Lines | Purpose |
|--------|-------|---------|
| `calib_linear.py` | 456 | Linear initialization: Procrustes (MeTRAbs) or collinearity/coplanarity (RTMPose) |
| `ba.py` | 621 | Bundle Adjustment: confidence-weighted NLL, bone constraints, sparse Jacobian |

### Postprocessing (`postprocessing/`)

| Module | Lines | Purpose |
|--------|-------|---------|
| `evaluate_calibration.py` | 256 | MRE computation, best/worst frame visualization, TOML export |
| `scale_scene.py` | 277 | Ground plane fitting, gravity alignment, metric scaling by person height |
| `visualize_results.py` | 546 | 3D GIF/MP4 with skeleton, camera frustums, live MRE overlay |

### Scripts (`scripts/`)

| Module | Lines | Purpose |
|--------|-------|---------|
| `calibrate.sh` | 490 | Main 7-step bash orchestrator |
| `run_calib_linear.py` | 266 | 1000-frame chunking, best-chunk selection by MRE |
| `detect_outlier_frames.py` | 187 | Per-camera outlier detection with dual threshold |
| `run_ba.py` | 88 | BA wrapper with OOM auto-retry (frame_skip += 5) |

## Legacy Archive

The `legacy/archive/` directory contains 27 files from older pipeline versions:

| Category | Files | Status |
|----------|-------|--------|
| RANSAC calibration | `calib_ransac.py/.sh` | Replaced by Procrustes + BA |
| Dataset-specific | `calib_synadl.py`, `prepare_*.sh` | Research artifacts |
| Model retraining | `retrain.py/.sh` | Research only |
| Old visualization | `vis.py/.sh` | Replaced by `visualize_results.py` |
| Old evaluation | `eval.py/.sh` | Replaced by `evaluate_calibration.py` |
| Old runners | `run_h36m.sh`, `run_panoptic.sh`, etc. | Replaced by `calibrate.sh` |

## Key Configuration Files

| File | Purpose |
|------|---------|
| `conda_linux.yaml` | Conda environment (`human_calib`) with all dependencies |
| `config/config.yaml` | Auto-generated per-session metadata (cameras, joints, resolution) |
| `argument.py` | Shared argparse definitions for all Python scripts |
| `.gitignore` | Excludes videos, models, data directories, credentials |
| `.gitmodules` | Git submodule configuration for VideoPose3D |
