# CLI Reference

## Main Entry Point

### `calibrate.sh`

```bash
bash scripts/calibrate.sh <video_dir> <calib_toml> [output_dir] [device] [mode] [options]
```

#### Positional Arguments

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `video_dir` | Yes | — | Folder containing synchronized video files |
| `calib_toml` | Yes | — | TOML file with camera intrinsics |
| `output_dir` | No | `./data/session_<timestamp>` | Output directory |
| `device` | No | `cuda` | `cuda` or `cpu` |
| `mode` | No | `balanced` | RTMPose model size: `lightweight`, `balanced`, `performance` |

#### Named Options

| Flag | Default | Description |
|------|---------|-------------|
| `--pose_engine <engine>` | `rtmpose` | `metrabs` (recommended) or `rtmpose` |
| `--height <meters>` | — | Subject height for metric scaling (enables step 8) |
| `--ref_frame <int>` | — | Frame where subject stands straight (for scaling) |
| `--start_frame <int>` | — | First video frame to process |
| `--end_frame <int>` | — | Last video frame to process |
| `--frame_skip <int>` | 10 | Subsample interval for calibration |
| `--conf_threshold <float>` | 0.5 | Minimum 2D keypoint confidence (0.0-1.0) |
| `--ref_cam <int>` | auto | 1-indexed camera ID for Procrustes reference |
| `--auto_outlier_drop` | true | Enable per-camera outlier frame detection |
| `--no_auto_outlier_drop` | — | Disable outlier detection |
| `--outlier_abs_px <float>` | 50 | Absolute reprojection threshold (pixels) |
| `--outlier_x_median <float>` | 5 | Relative threshold multiplier (x median error) |
| `--save_video` | off | Generate overlay video (RTMPose only) |

#### Examples

```bash
# Demo with MeTRAbs (recommended)
bash scripts/calibrate.sh \
    demo demo/Calib_scene.toml output/demo_metrabs \
    --pose_engine metrabs --height 1.78 --ref_frame 5

# RTMPose with frame range
bash scripts/calibrate.sh \
    input/my_session input/my_session/Calib_scene.toml output/my_session \
    cuda performance \
    --start_frame 100 --end_frame 500 --frame_skip 5

# Quick test on CPU
bash scripts/calibrate.sh \
    demo demo/Calib_scene.toml output/demo_cpu cpu
```

---

## Individual Scripts

### `pose/metrabs_inference.py`

```bash
conda run -n metrabs_opensim python pose/metrabs_inference.py \
    --video_dir <dir> --calib_toml <toml> --output_dir <dir> \
    --aid <int> --pid <int> --gid <int> \
    [--subset_name noise_1_0] [--batch_size 8] \
    [--start_frame <int>] [--end_frame <int>]
```

### `pose/rtmlib_inference.py`

```bash
python pose/rtmlib_inference.py \
    --video_dir <dir> --output_dir <dir> \
    --aid <int> --pid <int> --gid <int> \
    [--subset_name noise_1_0] [--device cuda] [--mode balanced] \
    [--start_frame <int>] [--end_frame <int>] [--save_video]
```

### `pose/inference.py` (VideoPose3D)

```bash
python pose/inference.py \
    --prefix <output_dir> --aid <int> --pid <int> --gid <int> \
    --target noise_1_0 --dataset MyDataset \
    --model pretrained_h36m_detectron_coco.bin --device cuda
```

### `scripts/run_calib_linear.py`

```bash
python scripts/run_calib_linear.py \
    [--conf_threshold 0.5] [--ref_cam <int>] \
    <output_dir> <aid> <pid> <gid> <subset> <frame_skip> <dataset>
```

### `scripts/detect_outlier_frames.py`

```bash
python scripts/detect_outlier_frames.py \
    --prefix <output_dir> --subset noise_1_0 \
    --aid <int> --pid <int> --gid <int> \
    --calib linear_1_0 --video_dir <dir> \
    [--abs_px 50] [--x_median 5] [--conf_threshold 0.5]
```

### `scripts/run_ba.py`

```bash
python scripts/run_ba.py \
    <output_dir> <aid> <pid> <gid> <frame_skip> \
    <lambda1> <lambda2> <target> <dataset> \
    <obs_mask> <save_obs_mask> [conf_threshold]
```

### `postprocessing/evaluate_calibration.py`

```bash
python postprocessing/evaluate_calibration.py \
    --prefix <output_dir> --calib <calib_name> \
    [--visualize] [--export_toml <filename>] \
    [--conf_threshold 0.5]
```

### `postprocessing/scale_scene.py`

```bash
python postprocessing/scale_scene.py \
    --prefix <output_dir> --calib <calib_name> \
    --height <meters> --frame_idx <int> \
    [--input_toml <toml>] [--export_toml <filename>] \
    [--video_dir <dir>] [--pose_engine metrabs]
```

### `postprocessing/visualize_results.py`

```bash
python postprocessing/visualize_results.py \
    --prefix <output_dir> --calib <calib_name> \
    [--output <filename.gif>] [--export_trc <filename.trc>] \
    [--conf_threshold 0.5]
```

### `tools/create_cameras_from_toml.py`

```bash
python tools/create_cameras_from_toml.py \
    --toml_path <calib.toml> --output_dir <dir> \
    --gid <int> --cam_names <name1> <name2> ...
```

---

## Shared Arguments (`argument.py`)

These arguments are defined centrally and shared across scripts:

| Argument | Type | Default | Used By |
|----------|------|---------|---------|
| `--dataset` | str | SynADL | All calibration scripts |
| `--aid` | int | 23 | All scripts |
| `--pid` | int | 102 | All scripts |
| `--gid` | int | 3 | All scripts |
| `--cid` | int | 21 | Individual camera scripts |
| `--frame_skip` | int | 15 | Linear calib, BA |
| `--prefix` | str | ./third_party/SynADL/ | All scripts |
| `--device` | str | cuda | Pose extraction, lifting |
| `--pose_engine` | str | rtmpose | calibrate.sh, scale_scene |
| `--conf_threshold` | float | 0.5 | All calibration scripts |
| `--ref_cam` | int | None | Linear calibration |
| `--frame_start` | int | None | Pose extraction |
| `--frame_end` | int | None | Pose extraction |

Note: `calibrate.sh` overrides many defaults when calling subscripts.
