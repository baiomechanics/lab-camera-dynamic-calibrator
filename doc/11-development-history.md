# Development History

## Evolution Timeline

### Phase 1: Cleanup (February 2026)

**Commit:** `ca63bca` — "chore: phase 1 cleanup — remove dead code, archive legacy scripts"

- Moved 27 legacy scripts to `legacy/archive/`
- Removed dead code paths and unused imports
- Archived RANSAC-based calibration method
- Archived dataset-specific runners (H36M, Panoptic, GAFA, SynADL)

### Phase 2: Reorganization (February 2026)

**Commit:** `3c383cf` — "chore: phase 2 reorganization — group code into thematic subfolders"

- Created `pose/`, `calibration/`, `postprocessing/`, `scripts/` directories
- Moved files from flat layout into thematic folders
- Maintained backward compatibility during transition

### Phase 3: Core Library (February 2026)

**Commit:** `b360a9a` — "chore: phase 3 — split monolithic util.py into core/ subpackage"

- Broke apart `util.py` (~800 lines) into:
  - `core/skeletons.py` — Joint and bone definitions
  - `core/geometry.py` — Triangulation and projection math
  - `core/poses_io.py` — JSON I/O operations
  - `core/filtering.py` — Visibility and orientation helpers
  - `core/gpu.py` — GPU selection

### Phase 4: Python Wrappers (March 2026)

**Commit:** `f3ee264` — "chore: phase 4 — convert wrapper bash scripts to Python"

- Replaced bash wrappers with Python scripts:
  - `run_calib_linear.py` (chunked calibration)
  - `detect_outlier_frames.py` (auto outlier detection)
  - `run_ba.py` (BA with OOM retry)
- Better error handling and argument passing

### MeTRAbs Integration (February-April 2026)

**Commit:** `52efea1` — "feat: MeTRAbs integration, BA optimization, and pipeline overhaul"

Major architectural shift:
- Added MeTRAbs as primary pose backend (metric 3D)
- Implemented Procrustes alignment path in `calib_linear.py`
- Added auto reference-camera selection
- Introduced confidence weighting in bundle adjustment
- Added sparse Jacobian computation
- Added auto-balanced lambda2 for bone regularization

### Production Hardening (April 2026)

**Commit:** `ca048a9` — "fix(calibrate): validate --ref_frame vs [--start_frame, --end_frame]"

- Input validation for frame range consistency
- Silenced TensorFlow startup noise

**Commit:** `86096c9` — "feat(metrabs): production-ready 87-joint calibration pipeline"

- Full 87-joint extraction with quality filtering
- Dark frame detection, collapsed skeleton rejection
- Savitzky-Golay smoothing
- Dropped frame sidecar support

### Visualization & Documentation (May 2026)

**Commit:** `177c06b` — "feat(viz): light theme + live MRE metrics overlay; docs refresh"

- Light theme for visualizations
- Live MRE metrics card in 3D animation
- Per-camera MRE breakdown in overlay

**Commits:** `daf87e3`, `9d5cceb`, `4fbdfd5` — Documentation updates

- Refreshed README and HOWTO
- Updated graphical abstract
- Fixed stale references (util.py -> core/skeletons.py)

## Architecture Decisions

### Why Two Pose Engines?

**RTMPose + VideoPose3D** was the original pipeline. It works but:
- Two-step process (2D detection + 3D lifting) introduces error accumulation
- VideoPose3D outputs relative-scale 3D (not metric)
- 25 joints with 12 bones provides fewer geometric constraints
- Result: ~8.5 px MRE

**MeTRAbs** was integrated as the recommended path because:
- Single forward pass → 2D + 3D metric simultaneously
- Procrustes initialization is more robust than collinearity
- 26 calibration joints with 27 bones provides richer constraints
- Metric scale enables direct comparison across cameras
- Result: ~3.5 px MRE (2.4x improvement)

### Why Chunked Linear Calibration?

- Visibility varies across the sequence (person moves in/out of views)
- 1000-frame chunks naturally isolate segments with good multi-view coverage
- Best-chunk selection provides robustness to local visibility gaps
- Reduces memory footprint for large sequences

### Why Auto-Balanced Lambda2?

With MeTRAbs metric 3D, bone lengths are already consistent before optimization. A fixed lambda2 would either:
- Be too large: over-constrain and slow convergence
- Be too small: provide no regularization benefit

Auto-balancing targets bone term at ~10% of NLL energy — meaningful but not dominant.

### Why Sparse Jacobians?

Bundle adjustment scales poorly with dense Jacobians:
- Dense: O(N^2 * C^2 * J^2) memory, O(N^3) per iteration
- Sparse: O(N * C * J) memory, 50-200x speedup

Each reprojection residual depends on only 9 parameters (6 camera + 3 point), making the Jacobian naturally sparse.

## Contributor

- **Florian Delaplace** — Primary maintainer, all major features
- **Original work:** Based on research by Sang-Eun Lee (2022)
