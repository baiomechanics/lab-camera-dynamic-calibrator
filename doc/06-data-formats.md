# Data Formats & I/O

## Input Formats

### Video Files

- **Formats:** MP4, AVI, MOV
- **Requirement:** Synchronized across cameras (same start time, same frame count)
- **Naming:** Files sorted alphabetically → mapped to camera IDs (cam01.mp4 → C001, cam02.mp4 → C002, ...)

### Camera Intrinsics (TOML)

Pose2Sim/AniPose format:

```toml
[cam01]
name = "cam01"
size = [1920, 1080]
matrix = [[1500.0, 0.0, 960.0], [0.0, 1500.0, 540.0], [0.0, 0.0, 1.0]]
distortions = [0.0, 0.0, 0.0, 0.0, 0.0]
rotation = [0.0, 0.0, 0.0]       # Rodrigues vector (optional, for initial extrinsics)
translation = [0.0, 0.0, 0.0]    # (optional, for initial extrinsics)

[cam02]
...
```

### Dropped Frame Sidecars

Optional `.dropped.json` files alongside videos:

```json
{
  "dropped_frame_indices": [15, 42, 117, 203]
}
```

Created by pipeline sync tools or by `detect_outlier_frames.py`. Persists across re-runs.

---

## Internal Formats

### Pose JSON

Used for 2D and 3D pose data per camera. Created by pose extraction, consumed by calibration.

```json
{
  "data": [
    {
      "frame_index": 0,
      "skeleton": [
        {
          "pose": [x1, y1, x2, y2, ..., xN, yN],
          "score": [s1, s2, ..., sN]
        }
      ]
    },
    {
      "frame_index": 1,
      "skeleton": [
        {
          "pose": [x1, y1, x2, y2, ..., xN, yN],
          "score": [s1, s2, ..., sN]
        }
      ]
    }
  ]
}
```

- **2D pose:** `pose` is flattened (N_joints * 2) with [x, y] pixel coordinates
- **3D pose:** `pose` is flattened (N_joints * 3) with [x, y, z] in millimeters (MeTRAbs) or relative units (VideoPose3D)
- **score:** Per-joint confidence [0.0, 1.0]. Zero means excluded from calibration.
- **frame_index:** Original video frame number (preserves mapping to source)

### Camera JSON

Stores camera calibration parameters for the session.

```json
{
  "CAMID": [1, 2, 3, 4],
  "K": [
    [[1500.0, 0.0, 960.0], [0.0, 1500.0, 540.0], [0.0, 0.0, 1.0]],
    ...
  ],
  "R_w2c": [
    [[1.0, 0.0, 0.0], [0.0, 1.0, 0.0], [0.0, 0.0, 1.0]],
    ...
  ],
  "t_w2c": [
    [[0.0], [0.0], [0.0]],
    ...
  ],
  "dist_coeffs": [
    [0.0, 0.0, 0.0, 0.0, 0.0],
    ...
  ]
}
```

- **K:** 3x3 intrinsic matrix per camera
- **R_w2c:** 3x3 world-to-camera rotation matrix per camera
- **t_w2c:** 3x1 world-to-camera translation vector per camera
- **dist_coeffs:** 5-parameter distortion [k1, k2, p1, p2, k3]

### Skeleton World JSON

Reference 3D skeleton in world coordinates.

```json
{
  "skeleton": [[[x, y, z], ...], ...],
  "frame_indices": [0, 1, 2, ..., N]
}
```

- **skeleton:** Shape (N_frames, N_joints, 3) — world 3D positions
- **frame_indices:** Mapping from array index to video frame number

### Config YAML

Auto-generated session metadata:

```yaml
MyDataset:
  available_joints: [0, 1, 2, ..., 25]
  camera_ids: [1, 2, 3, 4]
  frame_rate: 30
  height: 1080
  width: 1920
  scale: 1
  ransac_th_2d: 200.0
  ransac_th_3d: 1.0
aid: 1
pid: 1
gid: 1
```

---

## Output Formats

### Calibrated TOML

Final output with updated extrinsics in Pose2Sim format:

```toml
[cam01]
name = "cam01"
size = [1920, 1080]
matrix = [[1500.0, 0.0, 960.0], [0.0, 1500.0, 540.0], [0.0, 0.0, 1.0]]
distortions = [0.0, 0.0, 0.0, 0.0, 0.0]
rotation = [rx, ry, rz]       # Updated Rodrigues vector
translation = [tx, ty, tz]    # Updated, metric-scaled
```

### TRC File (OpenSim)

Tab-separated values with 3D skeleton for biomechanical analysis:

```
PathFileType    4    (X/Y/Z)    skeleton_3d.trc
DataRate    CameraRate    NumFrames    NumMarkers    Units    OrigDataRate    OrigDataStartFrame    OrigNumFrames
30    30    N    26    mm    30    1    N
Frame#    Time    joint1        joint2        ...
        X1    Y1    Z1    X2    Y2    Z2    ...
1    0.000    100.5    200.3    50.1    ...
```

### Visualization Outputs

- **GIF/MP4:** 3D skeleton animation with camera frustums and live MRE overlay
- **MRE visualizations:** Per-camera best/worst frame images showing reprojection error
- **BA convergence plot:** Cost history PNG with iteration count and timing

---

## Directory Layout (Runtime)

```
output/my_session/                          # OUTPUT_DIR
├── noise_1_0/                              # SUBSET
│   ├── 2d_joint/
│   │   ├── A001_P001_G001_C001.json       # 2D poses per camera
│   │   ├── A001_P001_G001_C002.json
│   │   └── ...
│   ├── 3d_joint/                           # MeTRAbs only
│   │   ├── A001_P001_G001_C001.json       # 3D poses per camera
│   │   └── ...
│   ├── 2d_joint_halpe26/                   # For scale_scene.py
│   │   └── ...
│   ├── cameras_G001.json                   # Intrinsics + initial extrinsics
│   └── skeleton_w_G001.json                # World skeleton reference
│
├── results/
│   ├── linear_1_0.json                     # Linear calibration result
│   ├── linear_1_0_ba.json                  # After bundle adjustment
│   ├── linear_1_0_ba_oriented_scaled.json  # After scaling
│   ├── Calib_scene_calibrated.toml         # Final TOML output
│   ├── camera/
│   │   ├── visu_3d_FINAL.gif              # 3D visualization
│   │   └── 3d_skeleton_FINAL.trc          # OpenSim skeleton
│   ├── MRE_visualizations/                 # Per-camera diagnostic images
│   └── ba_cost_live_iter*.png              # BA convergence plots
│
└── *.dropped.json                          # Per-video frame exclusion lists
```

## File Path Conventions

| Pattern | Meaning |
|---------|---------|
| `A{aid}` | Activity/Action ID (zero-padded 3 digits) |
| `P{pid}` | Person ID |
| `G{gid}` | Group/Scene ID |
| `C{cid}` | Camera ID (1-indexed) |
| `noise_1_0` | Subset name (legacy from noise augmentation experiments) |
| `linear_1_0` | Linear calibration with noise=1, scale=0 |
| `_ba` suffix | After bundle adjustment |
| `_oriented_scaled` suffix | After metric scaling and gravity alignment |

## Frame Index Mapping

Pose JSONs store original video frame numbers in `frame_index`. When `--start_frame` and `--end_frame` are used, a mapping translates between:

- **Video frame number:** The actual frame in the source video
- **JSON array index:** Position in the `data` array
- **Calibration index:** Position after visibility filtering

The `skeleton_w_G{gid}.json` file stores the complete `frame_indices` array for this mapping.
