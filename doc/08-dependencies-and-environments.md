# Dependencies & Environments

## Two Conda Environments

The pipeline requires two separate conda environments due to incompatible Python/TensorFlow/PyTorch versions.

### Environment 1: `human_calib` (Main)

**Definition:** `conda_linux.yaml`

Used for: Everything except MeTRAbs pose extraction.

```
Python 3.8
├── Deep Learning
│   ├── pytorch 1.13.1
│   ├── torchvision 0.14.1
│   ├── pytorch-cuda 11.7
│   └── mkl 2024.0.0            # Fixes iJIT_NotifyEvent symbol error
│
├── Pose Estimation
│   ├── rtmlib                   # RTMPose ONNX wrapper
│   └── onnxruntime-gpu          # ONNX inference backend
│
├── Computer Vision
│   └── opencv-contrib-python 4.7.0.72
│
├── Calibration & Math
│   ├── scipy                    # Bundle adjustment optimizer
│   ├── numpy                    # Array operations
│   ├── numba                    # JIT compilation for loops
│   └── pycalib-simple           # Camera calibration utilities
│
├── I/O & Configuration
│   ├── pyyaml                   # YAML config parsing
│   ├── tomli                    # TOML parsing (Python 3.8 compat)
│   ├── imageio 2.22.2           # Video I/O
│   └── portalocker              # File locking
│
├── Visualization
│   ├── matplotlib               # Plots and convergence charts
│   └── ffmpeg                   # Video encoding
│
├── Utilities
│   ├── pandas                   # Data manipulation
│   ├── tqdm                     # Progress bars
│   ├── networkx                 # Graph processing
│   ├── tabulate                 # Pretty table printing
│   ├── termcolor                # Colored terminal output
│   ├── nvgpu                    # GPU info scanning
│   └── ninja                    # Build system (for C extensions)
```

**Installation:**
```bash
conda env create -f conda_linux.yaml
conda activate human_calib
```

### Environment 2: `metrabs_opensim` (MeTRAbs)

**Setup:** Via separate repository https://github.com/flodelaplace/Metrabs_to_Opensim

Used for: MeTRAbs pose extraction only (step 1 when `--pose_engine metrabs`).

```
Python 3.10
├── TensorFlow 2.12.0
├── CUDA Toolkit 11.8
├── cuDNN 8.9
├── tensorflow-hub              # Auto-downloads metrabs_l model (~700 MB)
├── supervision 0.27+           # Person detection preprocessing
└── cameralib                   # Camera projection utilities
```

**Why separate?** PyTorch 1.13 (CUDA 11.7) and TensorFlow 2.12 (CUDA 11.8) have conflicting CUDA toolkit requirements. Running them in the same environment causes symbol resolution failures.

### Environment Switching

`calibrate.sh` handles this transparently:

```bash
# MeTRAbs step — switch to metrabs_opensim
conda run --live-stream -n metrabs_opensim python -u pose/metrabs_inference.py ...

# All other steps — run in current (human_calib) environment
python scripts/run_calib_linear.py ...
python scripts/run_ba.py ...
```

The `--live-stream` flag preserves tqdm progress bars that would otherwise be buffered.

## External Models

### VideoPose3D (RTMPose path only)

```bash
bash scripts/setup_models.sh
```

- Clones https://github.com/facebookresearch/VideoPose3D into `third_party/`
- Downloads pretrained weights: `pretrained_h36m_detectron_coco.bin`
- Not needed if using MeTRAbs path

### MeTRAbs Model

- Auto-downloaded by TensorFlow Hub on first run (~700 MB)
- Cached in `~/.cache/tfhub_modules/`
- Model ID: `metrabs_l` (large variant)

## Platform-Specific Notes

### Linux (Ubuntu 22.04)

Primary supported platform. No special setup beyond conda.

### WSL2

CUDA passthrough requires LD_LIBRARY_PATH fix (auto-applied by `calibrate.sh`):
```bash
export LD_LIBRARY_PATH=/usr/lib/wsl/lib:${LD_LIBRARY_PATH:-}
```

### GPU Selection

`core/gpu.py` auto-selects GPU:
1. Scans all GPUs for free memory
2. Picks GPU with < 18 MiB used (idle)
3. Falls back to GPU with most free memory
4. Sets `CUDA_VISIBLE_DEVICES` environment variable

Manual override: `--device cuda` or set `CUDA_VISIBLE_DEVICES` before running.

## Version Pinning Rationale

| Package | Version | Reason |
|---------|---------|--------|
| Python | 3.8 | Lowest common denominator for all dependencies |
| PyTorch | 1.13.1 | Last version with CUDA 11.7 support in conda |
| opencv-contrib | 4.7.0.72 | Includes SIFT and other contrib algorithms |
| imageio | 2.22.2 | API stability for video reading |
| mkl | 2024.0.0 | Fixes `undefined symbol: iJIT_NotifyEvent` runtime error |
| tomli | (latest) | Backport of tomllib for Python < 3.11 |
