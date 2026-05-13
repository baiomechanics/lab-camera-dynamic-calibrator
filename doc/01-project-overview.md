# Project Overview

## What Is This?

**lab-camera-dynamic-calibrator** is a production-ready Python/Bash pipeline for **extrinsic multi-camera calibration from human motion**. It uses human pose estimation to calibrate camera extrinsics without requiring checkerboards or calibration patterns — a person walking through the scene is enough.

## Key Idea

Instead of requiring a calibration board to be visible from all cameras simultaneously, this tool uses a human body as a calibration target. Since the human skeleton has known proportions and constraints (fixed bone lengths, articulated joints), observing the same person from multiple cameras provides enough geometric constraints to solve for camera positions and orientations.

## Two Pose Estimation Backends

| Backend | Type | Joints | Scale | Accuracy | Recommended |
|---------|------|--------|-------|----------|-------------|
| **MeTRAbs** | Single-pass 3D | 87 (mapped to 26) | Metric (mm) | ~3.5 px MRE | Yes |
| **RTMPose + VideoPose3D** | 2D detection + 3D lifting | 25 | Relative | ~8.5 px MRE | Legacy |

## Origin & License

- **License:** MIT (Copyright 2022 Sang-Eun Lee)
- **Primary maintainer:** Florian Delaplace
- **Repository:** https://github.com/flodelaplace/lab-camera-dynamic-calibrator

## Platform Support

| OS | Status | Notes |
|----|--------|-------|
| Linux (Ubuntu 22.04) | Fully supported | Recommended; tested with RTX 3500 Ada |
| Windows (WSL2) | Fully supported | CUDA via conda |
| macOS | Not supported | Requires NVIDIA GPU |

## Hardware Requirements

- NVIDIA GPU with CUDA support (RTX 3090, RTX 4090, RTX 3500 Ada, etc.)
- Recommended: 24+ GB VRAM for BA with frame_skip=1 on large scenes
- CPU fallback available but very slow (pose extraction only)

## Technology Stack

- **Python 3.8** (main pipeline)
- **Bash** (orchestration)
- **PyTorch 1.13** + CUDA 11.7 (pose models)
- **TensorFlow 2.12** (MeTRAbs, separate conda env)
- **SciPy** (bundle adjustment optimization)
- **OpenCV** (image/video processing)
- **Conda** (environment management)

## Codebase Stats

- **Total Python code:** ~7,600 lines
- **Main pipeline:** ~4,600 lines (excluding legacy)
- **Bash orchestrator:** ~490 lines
- **Legacy/archived:** ~2,500 lines (27 files)
