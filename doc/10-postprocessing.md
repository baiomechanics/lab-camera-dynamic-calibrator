# Postprocessing

## Evaluation (`postprocessing/evaluate_calibration.py`)

### Purpose

Quantifies calibration quality using Mean Reprojection Error (MRE) and generates diagnostic visualizations.

### How It Works

1. **Triangulate:** For each frame and joint, use DLT to reconstruct 3D position from multi-view 2D observations
2. **Reproject:** Project 3D points back to each camera's 2D space using calibrated extrinsics
3. **Measure error:** `MRE = mean(||observed_2d - reprojected_2d||)` in pixels

### Metrics Produced

| Metric | Scope | Description |
|--------|-------|-------------|
| Global MRE | All cameras, all frames | Single number quality indicator |
| Per-camera MRE | Per camera | Identifies problematic cameras |
| Best frame | Per camera | Frame with lowest reprojection error |
| Worst frame | Per camera | Frame with highest reprojection error |

### Diagnostic Outputs

- **Best/worst frame images:** Yellow lines drawn from observed 2D keypoints to reprojected positions. Short lines = good calibration. Saved to `MRE_visualizations/`.
- **TOML export:** Updated `Calib_scene.toml` with calibrated R, t values in Rodrigues vector format.

### Interpreting MRE

| MRE (pixels) | Quality | Notes |
|---------------|---------|-------|
| < 3 | Excellent | Sub-pixel accuracy |
| 3-5 | Good | Typical with MeTRAbs |
| 5-10 | Acceptable | May indicate one bad camera |
| > 10 | Poor | Check intrinsics, sync, visibility |

### Common Issues

- **High MRE on one camera only:** Check its intrinsic matrix (K) — focal length or principal point may be wrong
- **Low Procrustes residual but high MRE:** Intrinsics issue (the 3D shape is correct but projection fails)
- **All cameras high MRE:** Check video synchronization or confidence threshold

---

## Scene Scaling & Orientation (`postprocessing/scale_scene.py`)

### Purpose

Transforms calibration results into a metric, gravity-aligned coordinate system using a reference frame where the subject stands upright.

### Coordinate System Definition

```
        Y (up/gravity)
        │
        │    Z (forward)
        │   /
        │  /
        │ /
        └──────── X (right)
      Origin
   (feet center)
```

### Algorithm

**Step 1: Ground Plane Fitting**
- Extract foot keypoints (heels + toes) from reference frame
- Fit a plane through these 6-8 points (depending on skeleton format)
- Plane normal defines the approximate vertical direction

**Step 2: Axis Definition**
- **Y-axis (up):** Vector from head to ground centroid, normalized
- **X-axis (right):** Vector from right heel to left heel, orthogonalized against Y via Gram-Schmidt
- **Z-axis (forward):** Cross product X x Y (right-hand rule)

**Step 3: Metric Scaling**
- Transform head position into new coordinate frame
- Measure height: `measured_height = abs(head_Y)`
- Compute scale: `scale = user_height / measured_height`
- Apply scale to all camera translation vectors

### Joint Format Auto-Detection

| Detected joints | Format | Foot points used |
|-----------------|--------|------------------|
| 87 | bml_movi_87 | 8 (L/R heel, toe, foot, ankle) |
| 26 | MeTRAbs calib-26 | 8 (L/R heel, toe, foot, ankle) |
| 25 | OpenPose-25 | 6 (L/R BigToe, SmallToe, Heel) |

Can be forced via `--pose_engine metrabs|rtmpose`.

### Output

- `{calib}_oriented_scaled.json` — Transformed R_w2c and scaled t_w2c
- Optional TOML export with Rodrigues rotation vectors

---

## 3D Visualization (`postprocessing/visualize_results.py`)

### Purpose

Generates animated 3D visualizations of the calibrated skeleton and camera positions.

### What It Renders

```
┌────────────────────────────────────────┐
│              3D Scene                   │
│                                         │
│    △ cam1    △ cam2                     │
│      \       /                          │
│       \     /                           │
│     ╔══╧═══╧══╗  ← skeleton            │
│     ║  (o)    ║                         │
│     ║ /   \   ║                         │
│     ║/     \  ║                         │
│     ╚═══════╝                          │
│    △ cam3    △ cam4                     │
│                                         │
│  ┌─────────────────────────────┐       │
│  │ Cams: 4 │ Joints: 26       │       │
│  │ MRE: 3.5 px                │       │
│  │ cam01: 2.8  cam02: 3.1     │       │
│  │ cam03: 4.2  cam04: 3.9     │       │
│  └─────────────────────────────┘       │
└────────────────────────────────────────┘
```

### Features

- **Skeleton rendering:** Auto-detects format (87/26/25 joints), draws bones with appropriate topology
- **Camera frustums:** Pyramids showing position and viewing direction for each camera
- **MRE metrics card:** Per-frame overlay with mean MRE, per-camera MRE, visible camera count, detected joint count
- **Adaptive framing:** Percentile-based bounding box that tracks skeleton motion
- **View angle:** 20 degrees elevation, -60 degrees azimuth

### Output Formats

| Format | Extension | Notes |
|--------|-----------|-------|
| GIF | `.gif` | Default, smaller file size |
| MP4 | `.mp4` | If output filename ends in .mp4 |

### TRC Export

Optionally exports OpenSim-compatible TRC file:
- Tab-separated with frame number, time, and (X, Y, Z) per joint
- Units: millimeters
- Compatible with OpenSim biomechanical analysis software

### Light Theme

Recent update (May 2026) switched to light theme for better readability in presentations and publications.
