# Documentation Index

Technical documentation for **lab-camera-dynamic-calibrator** — a multi-camera extrinsic calibration pipeline using human pose estimation.

## Documents

| # | Document | Description |
|---|----------|-------------|
| 01 | [Project Overview](01-project-overview.md) | What this project does, technology stack, platform support, hardware requirements |
| 02 | [Repository Structure](02-repository-structure.md) | Directory tree, module responsibilities, file purposes, legacy archive |
| 03 | [Pipeline Flow](03-pipeline-flow.md) | End-to-end flow diagram, step-by-step walkthrough, conda switching, error handling |
| 04 | [Pose Estimation](04-pose-estimation.md) | MeTRAbs and RTMPose backends, VideoPose3D lifting, skeleton formats, quality filtering |
| 05 | [Calibration Algorithms](05-calibration-algorithms.md) | Linear initialization (Procrustes/collinearity), bundle adjustment, sparse Jacobians, outlier detection |
| 06 | [Data Formats](06-data-formats.md) | JSON schemas, TOML format, TRC export, directory layout, file naming conventions |
| 07 | [CLI Reference](07-cli-reference.md) | All command-line arguments for calibrate.sh and individual scripts |
| 08 | [Dependencies & Environments](08-dependencies-and-environments.md) | Conda environments, package versions, external models, platform notes |
| 09 | [Core Library](09-core-library.md) | `core/` package API: skeletons, geometry, I/O, filtering, GPU selection |
| 10 | [Postprocessing](10-postprocessing.md) | Evaluation metrics, scene scaling, 3D visualization, TRC export |
| 11 | [Development History](11-development-history.md) | Evolution timeline, architecture decisions, contributor info |

## Quick Start

For usage instructions, see the main [HOWTO.md](../HOWTO.md) or [CLI Reference](07-cli-reference.md).

For understanding the calibration math, start with [Calibration Algorithms](05-calibration-algorithms.md).

For understanding the data flow, see [Pipeline Flow](03-pipeline-flow.md) and [Data Formats](06-data-formats.md).
