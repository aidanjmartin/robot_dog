# Project log

Machine: Dell Precision 7680 laptop, Intel Core i9-13950HX (32 threads), 62 GB RAM, NVIDIA RTX 4000 Ada Generation Laptop GPU (12 GB), driver 580.178.04 (CUDA 13.0), Ubuntu 24.04.5 LTS, kernel 7.0.0-38, system Python 3.12.3, gcc 13.3.0.

Exam 1 ran on a lab desktop with Ubuntu 26.04. This log names every difference from [ENGR6010_Exam1 `q2_sdk/LOG.md`](https://github.com/aidanjmartin/ENGR6010_Exam1/blob/90150da/q2_sdk/LOG.md) and [`q3_apps/LOG.md`](https://github.com/aidanjmartin/ENGR6010_Exam1/blob/90150da/q3_apps/LOG.md).

## Phase A: laptop setup

Date: 2026-10-06.

### System

| step | command | outcome |
| :--- | :--- | :--- |
| NVIDIA driver | `nvidia-smi` | already installed: driver 580.178.04, CUDA 13.0, RTX 4000 Ada Laptop GPU with 12 GB |
| system packages | `sudo apt install build-essential git ffmpeg portaudio19-dev` | success: build-essential 12.10, git 2.43.0, ffmpeg 6.1.1, portaudio19-dev 19.6.0. cmake 3.28.3 was already installed. |
| time sync | `sudo apt install chrony`, then `chronyc tracking` | success: chrony 4.5 replaces systemd-timesyncd; synchronized to a Canonical time server, 26 µs from NTP time |
| Miniconda | `curl -fLO .../Miniconda3-latest-Linux-x86_64.sh`, check SHA-256 against repo.anaconda.com, `bash Miniconda3-latest-Linux-x86_64.sh -b -p ~/miniconda3` | success: conda 26.7.1, checksum `e8b25b92...338b` matches, shell startup files left unchanged |

Differences from Exam 1: this setup drops `libglfw3` and `python3-venv`, which served the MuJoCo viewer and venv environments, because this project uses neither. Ubuntu 24.04 ships cmake 3.28 where Exam 1 had 4.2; CycloneDDS 0.10.x needs 3.16 or later.

### Upstream repositories

`git clone <url> ~/go2_deps/<name> && git -C ~/go2_deps/<name> checkout <commit>`, at the commits Exam 1 pinned:

| repository | branch | commit | commit date |
| :--- | :--- | :--- | :--- |
| eclipse-cyclonedds/cyclonedds | `releases/0.10.x` | `5041f35` | 2026-03-23 |
| unitreerobotics/unitree_sdk2_python | default | `814556d` | 2026-09-21 |
| legion1581/unitree_webrtc_connect | default | `e0abc57` | 2026-08-16 |

On 2026-10-06, `5041f35` is still the head of `releases/0.10.x` (GitHub compare API). This setup skips Exam 1's `unitree_mujoco`, `unitree_rl_gym`, `rsl_rl`, Go2Py and `tfoldi/go2-webrtc`: the project uses no simulator, and `unitree_webrtc_connect` replaces go2-webrtc.

### CycloneDDS

| step | command | outcome |
| :--- | :--- | :--- |
| first build | `cmake .. -DCMAKE_INSTALL_PREFIX=../install -DBUILD_EXAMPLES=OFF && cmake --build . --target install` | success: library version 0.10.5; links the system OpenSSL 3.0.13, which later breaks WebRTC (see Problems) |
| rebuild without OpenSSL | `rm -rf build install`, then the same with `-DENABLE_SSL=OFF -DENABLE_SECURITY=OFF` | success: `ldd libddsc.so` lists no `libssl` or `libcrypto` |

`ENABLE_SSL` adds TLS over TCP to DDS, and `ENABLE_SECURITY` adds the DDS Security plugins (authentication and encryption). Exam 1's DDS reads from the robot used plain UDP and neither feature. Phase C confirms whether the robot accepts DDS from this build.

### Environments

Exam 1 used two environments, Python 3.11 for the SDK and Python 3.10 for Go2Py and WebRTC. Python 3.10 served Go2Py, which this project drops, so one robot environment holds both the SDK and the WebRTC library. It also holds PyTorch, because the safety filter runs the learned ensemble inside the control loop.

| step | command | outcome |
| :--- | :--- | :--- |
| create robot environment | `conda create -p .venv-robot --override-channels -c conda-forge python=3.11 pip` | success: Python 3.11.17 |
| install the SDK | `CYCLONEDDS_HOME=~/go2_deps/cyclonedds/install pip install -e ~/go2_deps/unitree_sdk2_python` | success: unitree_sdk2py 1.0.1, cyclonedds 0.10.2, numpy 2.4.6, opencv-python 5.0.0.93 |
| install the WebRTC library | `pip install -e ~/go2_deps/unitree_webrtc_connect` | success: unitree_webrtc_connect 2.2.0, aiortc 1.15.0 |
| install PyTorch | `pip install torch==2.14.1 --index-url https://download.pytorch.org/whl/cu130` | success: torch 2.14.1+cu130, `torch.cuda.is_available()` is True |
| reinstall the DDS binding after the rebuild | `CYCLONEDDS_HOME=... pip install --force-reinstall --no-deps --no-cache-dir cyclonedds==0.10.2` | success |
| check | `pip check`; `import unitree_sdk2py, cyclonedds, unitree_webrtc_connect, ssl, torch` in both import orders | no broken requirements; both orders import; Python uses conda's OpenSSL 3.6.5 |
| DDS on this computer | `example/helloworld/subscriber.py` and `publisher.py`, unchanged, run with `python -u` | success: the subscriber received all 30 messages. The examples name no interface, so this run used the Wi-Fi interface, not `lo`. |
| create ML environment | `conda create -p .venv-ml --override-channels -c conda-forge python=3.11 pip` | success: Python 3.11.17 |
| install PyTorch | `pip install torch==2.14.1 --index-url https://download.pytorch.org/whl/cu130` | success: torch 2.14.1+cu130 |
| install RL packages | `pip install gymnasium==1.4.0 stable-baselines3==2.9.0` | success; versions are the current PyPI releases on 2026-10-06 |
| check | `pip check`; `torch.cuda.is_available()`; a 4096 x 4096 matrix product on the GPU | no broken requirements; True; product runs |

The cu130 build of PyTorch needs an NVIDIA driver that supports CUDA 13.0, which driver 580 does. The two environments take 6.0 GB and 5.5 GB, mostly two copies of PyTorch with CUDA.

Phase A done check: `nvidia-smi` shows the GPU, `torch.cuda.is_available()` is True in both environments, and `unitree_sdk2py` and `unitree_webrtc_connect` both import. No step in Phase A talked to the robot.

## Phase B: project repository

Date: 2026-10-06.

| step | command | outcome |
| :--- | :--- | :--- |
| create the repository | `git init -b main` in `~/robot_dog`; folders `robot/`, `data/`, `models/`, `perception/`, `rl/`, `filter/`, `tracking/`, `eval/`, `scripts/` | success |
| copy from Exam 1 (commit `90150da`) | `.gitignore`, `.env.example`, `q3_apps/probe_signaling.py` and `q3_apps/webrtc_read_state.py` into `robot/` | success; each file starts with a line naming its source file and commit |
| adjust `.gitignore` | remove `data/` from Exam 1's recordings rules; add `.venv*` | `git check-ignore` ignores `.env`, `.env.local`, both environments and `recordings/`, and tracks `data/` and `.env.example` |

Exam 1's `.gitignore` ignored every `data/` folder, which would have hidden this project's `data/` code folder. Recorded sessions go in `recordings/`, which stays ignored, as do `*.jsonl`, `*.npz` and `*.mp4` anywhere.

`robot/probe_signaling.py` keeps Unitree's documented default addresses as fallbacks after `.env`: the wired main board, the robot's own hotspot, and the EDU's Jetson. They are Unitree's published defaults, not this robot's identity. `robot/webrtc_read_state.py` reads `../.env` relative to its own folder, which from `robot/` is the repository root, as before.

## Problems found

**CycloneDDS and conda's Python load two OpenSSL versions.** With CycloneDDS built at its default options (`ENABLE_SSL=AUTO`, `ENABLE_SECURITY=ON`), `libddsc.so` links Ubuntu 24.04's system `libcrypto.so.3` (OpenSSL 3.0.13). Importing `unitree_sdk2py` first loads that library, and then `import unitree_webrtc_connect` fails inside aioice's `import ssl` with `ImportError: /lib/x86_64-linux-gnu/libcrypto.so.3: version 'OPENSSL_3.3.0' not found`, because conda's Python needs its own, newer `libcrypto.so.3`, which has the same file name. Importing the WebRTC library first works, but the control loop needs both in one process, so import order is not a fix. Rebuilding CycloneDDS with `-DENABLE_SSL=OFF -DENABLE_SECURITY=OFF` removes its OpenSSL link. Exam 1 likely did not hit this because Ubuntu 26.04 ships a newer OpenSSL.

**The helloworld examples cut off and buffer output.** The publisher sends 30 messages, one per second. A first run with a 15 s timeout stopped it early, and the subscriber's buffered output was lost when its timeout ended it. Running both with `python -u` and a 40 s timeout gave the result above.
