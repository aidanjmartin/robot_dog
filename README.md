# robot_dog

Aidan J. Martin, ENGR 6010: AI in Robotics, Middle Tennessee State University, Fall 2026.

**Learned-Model Safety Filtering for Vision-Guided Reinforcement Learning Navigation on the Unitree Go2.**

A reinforcement learning (RL) policy drives a Unitree Go2 X to an object named by a text prompt while it avoids obstacles. Before the robot executes each command, a safety filter predicts the next 0.5 s of motion with a learned dynamics model and replaces any command that it predicts will violate obstacle clearance. The project asks whether a learned model of the Go2's command-to-motion dynamics can make an RL policy safe on real hardware, and whether that safety depends on model quality.

This repository builds on [ENGR6010_Exam1](https://github.com/aidanjmartin/ENGR6010_Exam1), which first reached this robot over its wired network. Every file copied from Exam 1 names its source file and commit on its first line. [`LOG.md`](LOG.md) records the machine setup, every test and the problems found.

## Terms

- **Go2 X**: the Go2 model used here. It accepts body velocity commands and Unitree's predefined modes, not joint-level commands.
- **DDS** (Data Distribution Service): the publish-subscribe middleware that Unitree's SDK uses over the robot's wired network, through the CycloneDDS implementation.
- **WebRTC** (Web Real-Time Communication): the browser protocol that the Unitree phone app uses. It carries the camera video and a data channel of named topics.
- **Sport mode**: Unitree's onboard gait controller. It turns body velocity commands into leg motion.

## Layout

```
README.md       this file
LOG.md          setup, tests and problems found, per phase
.env.example    names of the robot settings; copy to .env (untracked) and fill in
robot/          connection, command sending, and safety stop logic
data/           command generators and session logging
models/         physics baseline and learned ensemble
perception/     segmentation, calibration, LiDAR projection
rl/             Gymnasium environment and PPO training
filter/         safety filter
tracking/       overhead camera AprilTag tracking
eval/           metrics and figures
scripts/        setup and network scripts
```

Recorded sessions go in `recordings/`, which git ignores. Datasets stay out of the repository.

## Setup

[`LOG.md`](LOG.md) gives the full install order. In short:

- Two conda environments from conda-forge, created as prefixes in this folder and ignored by git:
  - `.venv-robot` (Python 3.11): `unitree_sdk2py`, `unitree_webrtc_connect` and PyTorch with CUDA
  - `.venv-ml` (Python 3.11): PyTorch with CUDA, Gymnasium and Stable-Baselines3
- Upstream repositories cloned to `~/go2_deps/` at pinned commits. CycloneDDS 0.10.x is built from source there, without OpenSSL.
- The robot address and any keys go in `.env`, never in tracked files.

## Robot scripts

Run each from the repository root. Each reads `.env`.

```bash
.venv-robot/bin/python robot/probe_signaling.py           # which WebRTC handshake the robot uses
.venv-robot/bin/python robot/webrtc_read_state.py --seconds 20   # camera and state over WebRTC, read-only
```
