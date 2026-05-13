# Pose Estimation

## Two Backend Engines

The pipeline supports two pose estimation backends. MeTRAbs is recommended for new work.

## MeTRAbs Pipeline (`pose/metrabs_inference.py`)

### Overview

MeTRAbs (Metric-Scale Truncation-Robust Heatmaps for Absolute 3D Pose) produces **metric 3D poses directly from images** in a single forward pass. It outputs 87 joints in the bml_movi_87 skeleton format, which are then mapped to a 26-joint calibration subset.

### Environment

Runs in a separate conda environment `metrabs_opensim`:
- Python 3.10
- TensorFlow 2.12.0
- CUDA Toolkit 11.8, cuDNN 8.9
- tensorflow-hub (auto-downloads metrabs_l model ~700 MB)

### Processing Flow

```
Video Frame
    │
    ├── Sidecar Check (.dropped.json)
    │   └── Skip frame if index in dropped list
    │
    ├── Dark Frame Detection
    │   └── Skip if mean brightness < 15
    │
    ├── MeTRAbs Model Forward Pass
    │   ├── Input: RGB frame + camera intrinsics
    │   ├── Output: 87 joints (2D + 3D metric in mm)
    │   └── Batched: 8 frames per batch with tf.data prefetch
    │
    ├── Quality Filtering
    │   ├── Bounding box area < 0.5% of image → rejected
    │   ├── 2D joint spread < 20 px → collapsed skeleton, rejected
    │   └── Joints within 10 px of image edge → confidence × 0.1
    │
    └── Savitzky-Golay Smoothing
        ├── Window: 11 frames, order: 3
        ├── Applied to 3D and 2D trajectories independently
        └── Per-joint, per-coordinate
```

### Joint Mapping

**87-joint to 26-joint mapping** via `METRABS_BML87_INDICES`:
```
Index: [67, 0, 70, 3, 69, 68, 76, 84, 72, 80, 77, 85, 74, 82, 73, 81,
        75, 83, 71, 79, 78, 86, 21, 52, 23, 54]
```

The 26 calibration joints include:
- Head, backneck, sternum, thorax
- Shoulders, elbows, wrists, hands (L/R)
- Hips, knees, ankles, feet (L/R)
- Heel, toe markers (L/R)

### Halpe26 2D Export

A separate `BML87_TO_HALPE26` mapping converts to Halpe26 format for backward compatibility with `scale_scene.py`:
- Saved to `2d_joint_halpe26/` directory
- Used for ground plane fitting (foot keypoint extraction)

### Output Format

Three JSON files per camera:
```
2d_joint/A{aid}_P{pid}_G{gid}_C{cid}.json       # 87-joint 2D
3d_joint/A{aid}_P{pid}_G{gid}_C{cid}.json       # 87-joint 3D (metric, mm)
2d_joint_halpe26/A{aid}_P{pid}_G{gid}_C{cid}.json  # 26-joint 2D
```

JSON structure:
```json
{
  "data": [
    {
      "frame_index": 0,
      "skeleton": [{
        "pose": [x1, y1, ..., xN, yN],
        "score": [s1, ..., sN]
      }]
    }
  ]
}
```

### Performance

- ~2 minutes per camera on GPU
- ~3.5 px MRE on demo dataset after full pipeline

---

## RTMPose Pipeline (`pose/rtmlib_inference.py`)

### Overview

RTMPose is a 2D-only keypoint detector using ONNX runtime. It detects Halpe26 (26 joints) which are then converted to OpenPose-25 format.

### Model Options

| Mode | Speed | Accuracy |
|------|-------|----------|
| `lightweight` | Fastest | Lower |
| `balanced` | Default | Good tradeoff |
| `performance` | Slowest | Highest |

### Processing Flow

```
Video Frame
    │
    ├── RTMPose BodyWithFeet Model
    │   ├── Detection: person bounding box
    │   ├── Pose: Halpe26 keypoints (26 joints)
    │   └── Select best person by bounding box area
    │
    └── Format Conversion
        ├── Halpe26 → OpenPose-25 (for VideoPose3D)
        └── Halpe26 saved separately (for scale_scene.py)
```

### Output Format

```
2d_joint/A{aid}_P{pid}_G{gid}_C{cid}.json          # OpenPose-25
2d_joint_halpe26/A{aid}_P{pid}_G{gid}_C{cid}.json   # Halpe26
```

---

## VideoPose3D Lifting (`pose/inference.py`)

### Overview

VideoPose3D lifts 2D poses to 3D using a temporal convolutional network. **Only used with RTMPose path** — skipped entirely with MeTRAbs.

### Architecture

- **Model:** `TemporalModel` — 5-layer temporal CNN
- **Filter widths:** [3, 3, 3, 3, 3]
- **Channels:** 1024
- **Receptive field:** 243 frames (121 frames lookahead/lookback)
- **Input:** COCO-17 joint format (converted from OpenPose-25)
- **Output:** H36M-17 3D positions (converted back to OpenPose-25)

### Processing Steps

1. Load 2D poses from `2d_joint/` (OpenPose-25 format)
2. Convert OpenPose-25 → COCO-17 via joint index mapping
3. Normalize 2D coordinates to [-1, 1] using screen dimensions
4. Pad sequence: 121 frames on each side (replicate edge frames)
5. Run temporal model on full sequence
6. Test-time augmentation: horizontal flip + average predictions
7. Convert H36M-17 → OpenPose-25 and save to `3d_joint/`

### Output

```
3d_joint/A{aid}_P{pid}_G{gid}_C{cid}.json
```

**Scale:** Relative (not metric) — requires post-calibration scaling.

---

## Skeleton Formats

### OpenPose-25 (RTMPose path)

**25 joints:** Nose, Neck, R/L Shoulder, R/L Elbow, R/L Wrist, R/L Hip, R/L Knee, R/L Ankle, R/L Eye, R/L Ear, R/L BigToe, R/L SmallToe, R/L Heel

**12 bones:** Major limb segments connecting torso to extremities.

### MeTRAbs Calib-26 (Primary format)

**26 joints:** Head, thorax, pelvis, backneck, sternum, R/L shoulders, R/L elbows, R/L wrists, R/L hands, R/L hips, R/L knees, R/L ankles, R/L feet, R/L heel, R/L toe

**27 bones:** Full body coverage including:
- 4-segment spine (pelvis → thorax → backneck → head)
- Articulated feet (ankle → heel, ankle → toe)
- Full arm chain (shoulder → elbow → wrist → hand)

### bml_movi_87 (MeTRAbs full output)

**87 joints:** 26 main joints + ~60 virtual landmarks from BML MoVi dataset.

Used internally by MeTRAbs. Remapped to calib-26 for calibration because virtual joints are linear combinations of main joints, making the full 87 rank-deficient for geometric constraints.

### Auto-Detection

`core/skeletons.py::get_bone_config(n_joints)` returns the appropriate bone topology:
- 87 joints → bml_movi_87 bones mapped to 87-joint indices
- 26 joints → MeTRAbs calib-26 topology
- 25 or other → OpenPose-25 topology

---

## Quality & Confidence

### Confidence Scoring

- **Per-joint confidence:** 0.0 to 1.0 from the pose model
- **Threshold:** `--conf_threshold` (default 0.5) — joints below are ignored
- **OOB penalty:** Joints within 10 px of image edge get confidence × 0.1
- **Outlier zeroing:** `detect_outlier_frames.py` zeros all scores for dropped frames

### Frame Dropping

Frames can be dropped at multiple stages:
1. **Sidecar files** (`.dropped.json`): Pre-existing frame exclusion lists
2. **Quality filters** (MeTRAbs): Dark, collapsed, no detection
3. **Outlier detection** (post-linear): High reprojection error frames
4. **BA outlier pass**: 2-pass optimization removes frames > 2x median error
