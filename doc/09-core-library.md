# Core Library (`core/`)

The `core/` package provides shared utilities used across the entire pipeline. All functions are re-exported from `core/__init__.py` for convenient importing.

## Skeletons (`core/skeletons.py`)

### Three Skeleton Formats

#### OpenPose-25 (`OP_KEY`)

```
Joints (25): Nose, Neck, RShoulder, RElbow, RWrist, LShoulder, LElbow,
             LWrist, MidHip, RHip, RKnee, RAnkle, LHip, LKnee, LAnkle,
             REye, LEye, REar, LEar, LBigToe, LSmallToe, LHeel,
             RBigToe, RSmallToe, RHeel

Bones (12): Neck-RShoulder, RShoulder-RElbow, RElbow-RWrist,
            Neck-LShoulder, LShoulder-LElbow, LElbow-LWrist,
            MidHip-RHip, RHip-RKnee, RKnee-RAnkle,
            MidHip-LHip, LHip-LKnee, LKnee-LAnkle
```

#### MeTRAbs Calib-26 (`METRABS_KEY`)

```
Joints (26): head, thorax, pelvis, rshoulder, relbow, rwrist, rhand,
             lshoulder, lelbow, lwrist, lhand, rhip, rknee, rankle,
             rfoot, lhip, lknee, lankle, lfoot, rheel, rtoe,
             lheel, ltoe, backneck, sternum, mhip

Bones (27): 4-segment spine, bilateral arms (3 each), bilateral legs (3 each),
            bilateral feet (2 each: ankle→heel, ankle→toe)
```

#### bml_movi_87

Full MeTRAbs output with 87 joints including ~60 virtual landmarks. Bone topology maps the 27-bone calib-26 structure onto 87-joint indices.

### Key Functions

```python
get_bone_config(n_joints)
```
Returns `(bone_array, active_joints)` based on detected joint count:
- 87 → bml_movi_87 bones
- 26 → MeTRAbs calib-26 bones
- Other → OpenPose-25 bones

### Joint Index Mapping

```python
METRABS_BML87_INDICES = [67, 0, 70, 3, 69, 68, 76, 84, 72, 80,
                          77, 85, 74, 82, 73, 81, 75, 83, 71, 79,
                          78, 86, 21, 52, 23, 54]
```
Maps 87-joint bml_movi to 26-joint calibration subset.

---

## Geometry (`core/geometry.py`)

### Triangulation

```python
triangulate(pt2d, P)
```
Basic DLT (Direct Linear Transform): 2D observations from multiple views → 3D point.
- Builds system `A @ X = 0` from projection matrices
- Solves via SVD (last row of Vt)

```python
triangulate_point(p_stack, proj_mat_stack, confs)
```
Weighted least-squares triangulation with per-joint confidence scores.

```python
triangulate_with_conf(p2d, s2d, K, R_w2c, t_w2c, mask)
```
Batch triangulation for full dataset: (cameras, frames, joints) → (frames, joints, 3).

### Projection

```python
project(K, R_w2c, t_w2c, pts3d_w)
```
3D world points → 2D pixel coordinates: `p2d = K @ (R @ X + t)`, then dehomogenize.

```python
project_cv2(K, R_w2c, t_w2c, pts3d_w, dist_coeffs)
```
OpenCV batch projection with lens distortion correction.

### Transforms

```python
invRT(R, t)  →  (R_inv, t_inv)
```
Invert world-to-camera transform: `R_inv = R^T`, `t_inv = -R^T @ t`.

```python
invRT_batch(R_list, t_list)  →  (R_inv_list, t_inv_list)
```
Batch inversion for all cameras.

### Constraint Building

```python
constraint_mat_from_single_view(R, normal, idx_v, idx_t)
```
Single-camera epipolar constraint for linear calibration.

```python
constraint_mat(R_list, normals_list, ...)
```
Stacks constraints from multiple views into sparse system.

```python
z_test_w2c(R1, t1, R2, t2, n1, n2)
```
Cheirality (positive depth) test: verifies triangulated points are in front of both cameras.

---

## Poses I/O (`core/poses_io.py`)

### Load Functions

```python
load_poses(filename)  →  (frame_indices, poses, scores)
```
Reads pose JSON. Returns:
- `frame_indices`: list of video frame numbers
- `poses`: ndarray (N, J*dim) flattened poses
- `scores`: ndarray (N, J) per-joint confidence

```python
load_eldersim_camera(filename)  →  (CAMID, K, R_w2c, t_w2c, dist_coeffs)
```
Reads camera JSON. Returns calibration arrays for all cameras.

```python
load_eldersim_skeleton_w(filename)  →  (skeleton, frame_indices)
```
Reads world skeleton reference JSON.

```python
load_eldersim(prefix, aid, pid, gid, subset, dataset)
```
Full loader: reads all cameras + all pose files + skeleton reference. Intersects common frames across cameras.

### Save Functions

```python
save_cam(filename, CAMID, K, R_w2c, t_w2c, dist_coeffs)
```
Saves camera calibration to JSON.

```python
save_joint(filename, frame_indices, poses, scores)
```
Saves pose data per camera to JSON.

---

## Filtering (`core/filtering.py`)

### Orientation Extraction

```python
joints2orientations(p3d, mask, bones)  →  ndarray (C, N', 3)
```
Extracts normalized bone direction vectors from 3D joint positions:
- Computes `bone_dir = normalize(joint[b1] - joint[b2])`
- Fills occluded joints with NaN
- Returns only visible bone directions

### Projection Extraction

```python
joints2projections(p2d, mask, joints)  →  ndarray (C, N', 2)
```
Extracts visible 2D joint projections per camera, filtering NaN entries.

Both functions are used by `calib_linear.py` to build geometric constraints.

---

## GPU Selection (`core/gpu.py`)

```python
select_gpu()  →  int
```

Auto-selects the best available NVIDIA GPU:
1. Scans all GPUs via `nvgpu` library
2. Selects GPU with < 18 MiB memory used (idle GPU)
3. If no idle GPU, selects one with most free memory
4. Sets `CUDA_VISIBLE_DEVICES` environment variable
5. Returns selected GPU index

Called at the start of pose extraction scripts.
