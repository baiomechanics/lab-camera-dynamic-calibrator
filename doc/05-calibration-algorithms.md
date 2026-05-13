# Calibration Algorithms

## Overview

Calibration happens in two stages:
1. **Linear initialization** — fast, closed-form solution for initial R, t estimates
2. **Bundle Adjustment** — non-linear refinement minimizing reprojection error

---

## Linear Initialization (`calibration/calib_linear.py`)

Two distinct algorithms depending on the pose engine:

### Path A: Procrustes Alignment (MeTRAbs)

Used when 3D metric poses are available (n_joints = 26 or 87).

#### Mathematical Formulation

**Goal:** Align all camera-local 3D poses to a reference camera's world frame.

For each camera pair (reference `ref`, source `src`), minimize:

```
||X_ref - s * R @ X_src - t||^2
```

Using the **Umeyama method**:
1. Center point clouds: `X_c = X - mean(X)`
2. Compute cross-covariance: `H = X_src_c^T @ X_ref_c / n`
3. SVD: `U, D, Vt = SVD(H)`
4. Rotation: `R = Vt^T @ S @ U^T` (with determinant correction: `S[2,2] = sign(det(Vt^T @ U^T))`)
5. Scale: `s = trace(D @ S) / var(X_src)`
6. Translation: `t = mean(X_ref) - s * R @ mean(X_src)`

#### Reference Camera Selection

Auto-selection via grid search:
```python
for each candidate camera:
    align all other cameras to candidate
    compute mean Procrustes residual
select camera with LOWEST mean residual
```

**Rationale:** The reference camera should have the most globally consistent 3D detections. Can be overridden with `--ref_cam`.

**Minimum constraint:** Requires >= 10 shared visible points per camera pair. Falls back to identity (R=I, t=0) if insufficient.

### Path B: Collinearity/Coplanarity (RTMPose)

Used when only 2D poses are available (2D-only initialization).

#### Step 1: Rotation via SVD

Stack per-camera unit direction vectors into matrix `V` (shape: N x 3C):
```
V = [v_cam1 | v_cam2 | ... | v_camC]
```

SVD: `Y, D, Zt = SVD(V)`

Extract scaled rotation: `R_all = sqrt(C) * Zt[:3, :]`
Normalize: force `R_0 = I` by left-multiplying inverse.

#### Step 2: Build Constraint Matrices

**Collinearity constraints** — each observed joint must lie on its camera ray:
```
normal @ R_w2c @ X_world + normal @ t_w2c = 0
```
(3 rows per constraint, 6 non-zeros)

**Coplanarity constraints** — for each camera pair, triangulation rays must be coplanar:
```
m = cross(n_a @ R_a, n_b @ R_b)
m @ R_a^T @ X_a + m @ t_a = 0
```

#### Step 3: Eigenvalue Problem

Stack into sparse matrix `C = [A; B]` and solve:
```
(C^T @ C) @ k = lambda * k
```

Extract 4 smallest eigenvectors (rank-4 null space: 3 rotation DOF + 1 scale).

#### Step 4: Translation & Scale Recovery

Last eigenvector provides translation direction. Scale from norm of first bone endpoint.

#### Step 5: Cheirality Correction

Test sign using first two cameras — flip t and X if triangulated points have negative depth.

### Visibility Filtering

Applied before both paths:
- Person must be visible (confidence > threshold) in **>= 2/3 of cameras** per frame
- If this yields < 20 frames, relax to minimum 2 cameras
- Bone orientation vectors normalized and converted to camera rays via `K^-T`

### Chunked Execution (`scripts/run_calib_linear.py`)

Splits frame range into 1000-frame chunks, runs calibration on each, selects best by MRE:

```
Total: 2500 frames
  Chunk 0: frames 0-999     → MRE = 4.2 px
  Chunk 1: frames 1000-1999 → MRE = 3.8 px  ← selected
  Chunk 2: frames 2000-2499 → MRE = 5.1 px
```

---

## Bundle Adjustment (`calibration/ba.py`)

### Cost Function

The optimization minimizes:

```
E = [e_nll ; lambda1 * e_var3d ; lambda2 * e_varbone]
minimize 0.5 * ||E||^2
```

### Component 1: Negative Log-Likelihood (NLL)

Confidence-weighted 2D reprojection error:

```
For each camera c, each visible joint j in frame f:
    x_proj = K[c] @ (R[c] @ X_world[f,j] + t[c])
    x_proj_2d = x_proj[:2] / x_proj[2]
    e = (x_observed - x_proj_2d) * sqrt(2) * confidence[c,f,j]
```

- Higher confidence → tighter residual tolerance
- Factor `sqrt(2)` normalizes chi-squared expectations
- Points below `conf_threshold` excluded entirely

### Component 2: Bone Direction Consistency (var3d)

Ensures all cameras measure the same bone directions in world frame:

```
For each bone b:
    For each camera c:
        d[c] = normalize(R[c] @ bone_direction_camera[c])
    var3d[b] = 1 - ||mean(d)||
```

- `1 - ||mean(d)||` measures directional spread (0 = perfect agreement)
- Depends only on rotation parameters (3C parameters)
- NaN-safe: ignores bones where endpoints are undetected

### Component 3: Bone Length Variance (varbone)

Enforces consistent bone lengths across the sequence:

```
For each bone b:
    lengths[f] = ||X[f, j1] - X[f, j2]|| for all frames f
    varbone[b] = var(lengths)
```

- Human skeleton has near-constant bone lengths
- Regularizes 3D point positions
- Auto-balanced lambda2 (see below)

### Auto-Balanced Lambda2

```python
nll_energy = sum(e_nll^2)
bone_energy = sum(e_bone^2)
target_ratio = 0.1  # bone term should be ~10% of NLL
lambda2 = sqrt(target_ratio * nll_energy / bone_energy)
lambda2 = min(lambda2, 1000.0)  # cap to prevent explosion
```

With MeTRAbs (metric 3D), bone variance is already small, so auto-scaling prevents wasted optimization effort.

### Sparse Jacobian

The Jacobian sparsity pattern is precomputed for 50-200x speedup:

| Component | Row count | Column dependencies |
|-----------|-----------|---------------------|
| NLL | 2 x (visible points) | 6 camera params + 3 point coords = 9 per point |
| var3d | B (bones) | 3C (all rotation params) |
| varbone | B (bones) | 6N (bone endpoints across all frames) |

Each reprojection residual (2 rows) touches exactly 9 columns. Typical sparsity: 0.01-0.1.

### Two-Pass Outlier Removal

```
Pass 1: Full optimization
    → Compute per-frame reprojection error
    → Mark frames with error > 2x median
    → Zero their confidence

Pass 2: Re-optimize with cleaned data
    → No further rejection
    → Final R, t, X output
```

### Convergence

- **Optimizer:** `scipy.least_squares` with Trust Region Reflective (TRF) method
- **Tolerances:** ftol=1e-7, xtol=1e-7, gtol=1e-7
- **Max evaluations:** min(max(60000, 4 * n_params), 80000)
- **Live plot:** Cost history PNG saved every 10 seconds

### OOM Auto-Retry (`scripts/run_ba.py`)

If bundle adjustment runs out of memory:
```
Attempt 1: frame_skip = 1  → OOM
Attempt 2: frame_skip = 6  → OOM
Attempt 3: frame_skip = 11 → Success (10% of data, ~10x less memory)
```

Maximum: frame_skip = 60, then abort.

---

## Outlier Detection (`scripts/detect_outlier_frames.py`)

### Dual Threshold Logic

A frame is flagged as outlier for camera `c` if BOTH conditions hold:
1. Reprojection error > `abs_px` (default: 50 pixels)
2. Reprojection error > `x_median` (default: 5) x per-camera median

### Actions on Outlier Frames

1. **Sidecar update:** Append frame indices to `<video>.dropped.json`
2. **Score zeroing:** Set all joint scores to 0.0 in pose JSONs
3. **Re-run linear:** Automatic re-calibration on cleaned data

This avoids re-running the expensive pose extraction step — only confidence is modified.

---

## Mathematical Glossary

| Term | Definition |
|------|-----------|
| **Extrinsics** | Camera pose in world frame: rotation R (3x3) and translation t (3x1) |
| **Intrinsics** | Camera internal parameters: focal length, principal point (K matrix) |
| **MRE** | Mean Reprojection Error — average pixel distance between observed and projected 2D points |
| **Procrustes** | Umeyama method for aligning 3D point clouds with rotation, translation, and scale |
| **DLT** | Direct Linear Transform — triangulation from 2D projections across views |
| **TRF** | Trust Region Reflective — bounded optimization method used by scipy |
| **NLL** | Negative Log-Likelihood — reprojection error weighted by detection confidence |
| **Cheirality** | Sign test ensuring triangulated 3D points are in front of (not behind) cameras |
| **Collinearity** | Constraint that a 3D point lies on a camera's projection ray |
| **Coplanarity** | Constraint that two camera rays and their baseline are coplanar (necessary for intersection) |
