# Pipeline Flow

## High-Level Overview

```
INPUT                           PIPELINE                              OUTPUT
──────────────                  ──────────────────────────            ──────────────
Synchronized videos  ──┐
                       ├──►  calibrate.sh (7-step orchestrator)  ──►  Calib_scene_calibrated.toml
Camera intrinsics    ──┘                                              3d_skeleton_FINAL.trc
(TOML)                                                                visu_3d_FINAL.gif
```

## Entry Point

```bash
bash scripts/calibrate.sh <video_dir> <calib_toml> [output_dir] [options]
```

The main orchestrator `calibrate.sh` drives the entire pipeline, calling Python scripts in sequence with proper conda environment switching.

## Step-by-Step Flow

### Step 1: Pose Extraction

**Purpose:** Extract 2D and 3D human pose from each camera's video.

```
VIDEO FILES (per camera)
        │
        ├── [MeTRAbs path - RECOMMENDED]
        │   └── metrabs_inference.py
        │       ├── conda env: metrabs_opensim (TF 2.12)
        │       ├── Model: MeTRAbs-L via TensorFlow Hub
        │       ├── Output: 87 joints (bml_movi_87), 2D + 3D metric
        │       ├── Quality filters: dark frames, collapsed skeleton, OOB
        │       └── Savitzky-Golay temporal smoothing
        │
        └── [RTMPose path - LEGACY]
            └── rtmlib_inference.py
                ├── conda env: human_calib
                ├── Model: RTMPose BodyWithFeet (ONNX)
                ├── Output: 26 joints (Halpe26), 2D only
                └── Three size modes: lightweight/balanced/performance
```

**Outputs created:**
- `{output}/noise_1_0/2d_joint/A{aid}_P{pid}_G{gid}_C{cid}.json` (per camera)
- `{output}/noise_1_0/3d_joint/...` (MeTRAbs only)
- `{output}/noise_1_0/2d_joint_halpe26/...` (both paths)

**Caching:** Skips extraction if output JSON files already exist with matching frame range.

### Step 2: Intrinsics Loading

**Purpose:** Convert camera intrinsics from TOML to JSON format.

```
Calib_scene.toml  ──►  create_cameras_from_toml.py  ──►  cameras_G{gid}.json
                                                          skeleton_w_G{gid}.json
```

- Matches camera names from TOML sections to video filenames (sorted alphabetically)
- Converts Rodrigues rotation vectors to rotation matrices
- Extracts K matrix, distortion coefficients per camera

### Step 3: Configuration Auto-Detection

**Purpose:** Update `config.yaml` with session-specific metadata.

Inline Python in `calibrate.sh` detects:
- Number of cameras (from video count)
- Number of joints (from 2D pose JSONs)
- Resolution, frame rate
- Camera ID list

Writes to `config/config.yaml` under `MyDataset` key.

### Step 4: 3D Lifting (RTMPose path only)

**Purpose:** Lift 2D poses to 3D using temporal convolution.

```
2d_joint/ (OpenPose-25)  ──►  inference.py (VideoPose3D)  ──►  3d_joint/
```

- **Skipped entirely with MeTRAbs** (already has metric 3D from step 1)
- Uses TemporalModel with 243-frame receptive field
- Test-time augmentation via horizontal flip

### Step 5: Linear Calibration

**Purpose:** Compute initial camera extrinsics (R, t) from pose data.

```
2d_joint/ + 3d_joint/ + cameras.json
        │
        ├── run_calib_linear.py (chunking)
        │   ├── Split into 1000-frame chunks
        │   ├── Run calib_linear.py on each chunk
        │   ├── Evaluate MRE per chunk
        │   └── Select best chunk
        │
        ├── [MeTRAbs] Procrustes alignment
        │   ├── Auto-select reference camera (lowest residual)
        │   ├── Umeyama method: align 3D point clouds
        │   └── Requires >= 10 shared visible points per camera pair
        │
        └── [RTMPose] Collinearity + coplanarity constraints
            ├── SVD-based rotation estimation
            ├── Eigenvalue problem for translation
            └── Cheirality correction
```

**Output:** `results/linear_1_0.json`

### Step 5b: Auto Outlier Detection (optional, on by default)

**Purpose:** Remove frames with poor pose estimates before bundle adjustment.

```
linear_1_0.json + 2d_joint/
        │
        └── detect_outlier_frames.py
            ├── Compute per-frame reprojection error per camera
            ├── Flag: error > abs_px (50) AND error > x_median (5x) * median
            ├── Write .dropped.json sidecars
            ├── Zero scores in pose JSONs
            └── Re-run linear calibration on cleaned data
```

### Step 6: Bundle Adjustment

**Purpose:** Non-linear refinement of camera extrinsics.

```
linear_1_0.json
        │
        └── run_ba.py (OOM auto-retry)
            └── ba.py (scipy.least_squares, TRF method)
                ├── Cost = NLL + lambda1*var3d + lambda2*varbone
                │   ├── NLL: confidence-weighted 2D reprojection
                │   ├── var3d: bone direction consistency across cameras
                │   └── varbone: bone length variance across frames
                ├── Sparse Jacobian (50-200x speedup)
                ├── Auto-balanced lambda2 (bone term ~ 10% of NLL)
                ├── 2-pass: optimize → reject outlier frames → optimize
                └── Live convergence plot (PNG every 10s)
```

**OOM retry logic:** If memory exhausted, increment `frame_skip` by 5 (up to 60).

**Output:** `results/linear_1_0_ba.json`

### Step 7: Evaluation & Visualization

**Purpose:** Compute quality metrics and generate visual output.

```
linear_1_0_ba.json
        │
        ├── evaluate_calibration.py
        │   ├── DLT triangulation from multi-view 2D
        │   ├── Per-camera MRE (Mean Reprojection Error)
        │   ├── Best/worst frame visualizations
        │   └── TOML export with updated R/t
        │
        └── visualize_results.py
            ├── 3D skeleton animation (GIF/MP4)
            ├── Camera frustum rendering
            └── Live MRE metrics overlay
```

### Step 8: Scaling & Orientation (optional)

**Purpose:** Transform to metric, gravity-aligned coordinates.

**Requires:** `--height` and `--ref_frame` arguments.

```
linear_1_0_ba.json + ref_frame
        │
        └── scale_scene.py
            ├── Fit ground plane from foot keypoints
            ├── Define Y-axis (vertical/gravity)
            ├── Define X-axis (left-right)
            ├── Scale by person height
            └── Export metric-aligned TOML
```

**Final outputs:**
- `results/Calib_scene_calibrated.toml` — metric extrinsics
- `results/camera/3d_skeleton_FINAL.trc` — OpenSim-compatible skeleton
- `results/camera/visu_3d_FINAL.gif` — 3D visualization with MRE overlay

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         calibrate.sh                                     │
│                                                                          │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐     │
│  │  Step 1       │     │  Step 2       │     │  Step 3              │     │
│  │  Pose Extract │────►│  Load Intrins │────►│  Config Auto-Detect  │     │
│  │  (MeTRAbs or  │     │  (TOML→JSON)  │     │  (config.yaml)       │     │
│  │   RTMPose)    │     │               │     │                      │     │
│  └──────────────┘     └──────────────┘     └──────────┬───────────┘     │
│                                                         │                │
│                                              ┌──────────▼───────────┐    │
│                                              │  Step 4 (RTMPose)    │    │
│                                              │  3D Lifting          │    │
│                                              │  (VideoPose3D)       │    │
│                                              └──────────┬───────────┘    │
│                                                         │                │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────▼───────────┐    │
│  │  Step 6       │     │  Step 5b      │     │  Step 5              │    │
│  │  Bundle       │◄────│  Outlier      │◄────│  Linear Calibration  │    │
│  │  Adjustment   │     │  Detection    │     │  (Chunked)           │    │
│  └──────┬───────┘     └──────────────┘     └──────────────────────┘    │
│         │                                                                │
│  ┌──────▼───────┐     ┌──────────────┐                                  │
│  │  Step 7       │────►│  Step 8       │                                  │
│  │  Evaluation & │     │  Scaling &    │                                  │
│  │  Visualization│     │  Orientation  │                                  │
│  └──────────────┘     └──────────────┘                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Conda Environment Switching

- **MeTRAbs step:** `conda run --live-stream -n metrabs_opensim python -u ...`
  - Uses `--live-stream` to preserve tqdm progress bars
  - Separate TensorFlow 2.12 + Python 3.10 environment
- **All other steps:** Run directly in `human_calib` environment (PyTorch + Python 3.8)
- **WSL2 fix:** `export LD_LIBRARY_PATH=/usr/lib/wsl/lib:${LD_LIBRARY_PATH:-}` (auto-applied)

## Error Handling

- `set -e` in bash: pipeline stops on any command failure
- Explicit check after linear calibration: exits if result file not found
- OOM auto-retry in BA: increments frame_skip by 5 up to maximum of 60
- Outlier detection: writes sidecar files and zeros scores without re-running extraction
