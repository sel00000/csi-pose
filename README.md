# csi-pose

**English** | [한국어](README.ko.md)

2D single-person pose estimation (18 joints) from WiFi CSI (Channel State
Information) alone, with rule-based fall detection on top. No camera or
wearables at inference time — the signal source is the trace a human body
leaves on 2.4GHz radio waves (per-subcarrier amplitude) through scattering
and shadowing.

This is a port of WiSPPN (Intel 5300 single NIC, 3×3 **antenna matrix**) onto
six off-the-shelf ESP32-S3 boards (3TX×3RX) forming a 3×3 **link matrix**.
A webcam+RTMPose teacher generates pseudo-labels; the student model regresses
joint coordinates from 50ms windows of CSI amplitude tensors — once trained,
no camera is needed.

<p align="center">
  <img src="figures/fall-demo.gif" alt="Real-time CSI pose + fall-detection demo" width="600"><br>
  <em>Real-time demo — the green 18-joint skeleton is inferred from WiFi CSI alone
  (no camera at inference). The top banner shows the fall detector firing:
  <strong>PRESENT | ALARM</strong>.</em>
</p>

## Pipeline

```
① firmware/  3 ESP32-S3 TX boards broadcast ESP-NOW beacons (~103pps); 3 RX
                boards extract CSI → 130B frames over serial
                (csi_link/ is the shared TX/RX component)
② host/      bridge: serial→raw-log preservation + MQTT relay · recorder: HDF5
                session writer · csi_pipe: clock-fit/alignment/sample-build
                library · tools: operations CLIs
③ teacher/   run RTMDet→RTMPose over the synchronously recorded webcam video
                to generate pose labels
④ train/     train the CSI→PAM regression network (WiSPPN-ESP)
⑤ rt/        real-time pose estimation (~20Hz) + fall-detection demo
```

## What it detects

The model outputs a continuous 18-joint 2D skeleton; two higher-level signals
are derived on top of it:

- **Posture — standing vs. lying down.** Classified from the aspect ratio of
  the estimated skeleton's core bounding box (roughly: box taller than wide =
  standing, wider than tall = lying). The standing→lying transition is also one
  of the fall cues below.
- **Falls.** A rule-based finite state machine (IDLE → IMPACT → ALARM). An
  IMPACT is raised when ≥2 of 3 cues fire — (R1) fast pelvis/hip descent,
  (R2) a standing→lying transition, (R3) the head dropping into the lower part
  of the frame — and is promoted to an ALARM only if a sustained "lying & still"
  posture is then confirmed over a hold window. Recovery (standing back up, or
  leaving the area) releases the alarm.

Replay-demo result: **10 of 11 staged falls detected, 2 false positives**
(single session).

> **Honest scope.** The fall thresholds are provisional, calibrated from a
> single session (`fall-demo-01`); quantitative lying-subset and cross-session
> evaluation are deferred to a fuller data campaign. The CSI-based stillness
> check is currently disabled (per-posture motion-energy distributions
> overlapped), so confirmation relies on pose geometry. Treat this as a working
> demo, **not a validated medical or safety device.**

## Key idea 1 — Time synchronization (aligning heterogeneous clocks)

The boards' esp_timer clocks are independent of each other and of the host.
Every capture (CSI packet arrival, webcam frame grab) is stamped with a
**single host clock, `time.time_ns()`** — but USB serial arrival times include
batching delay. The key idea is the **asymmetry** of that delay: USB delay can
only make packets late, never early. Therefore the **lower envelope (lower
convex hull)** of the (board time, host time) scatter is an unbiased estimate
of the true clock transform.

```
 ESP32-S3 board clock (esp_timer µs)      Host clock time.time_ns() — single reference
        │                                        │
        └────── USB serial arrival ──────► (esp, t_host) scatter plot
                                            │
              USB batching delay is +only   │   ← "arriving early" is physically impossible
                                            ▼
                       lower envelope of the scatter = true clock transform
            (offline: piecewise-linear fit per boot epoch / realtime: rolling-min)
                                            │
                                            ▼
        every CSI packet → corrected time t_fit → resampled onto a 100Hz (10ms) grid
                                            │
 webcam frame grab time (same host clock) ──┤
                                            ▼
              pairing correction: truth = stamp − correction (measured systematic
              delays, decomposed via STOP events / monitor-flip filming)
                                            ▼
              cut the trailing 50ms (5-packet) CSI window at each frame anchor → sample
```

Oscillator temperature drift is absorbed by windowed (600s) fits plus linear
interpolation; board reboots split epochs via boot_id. Residual uncertainty is
about ±15ms — equivalent to 3–5cm of label noise at fall speeds of 2–3m/s.

## Key idea 2 — Teacher labeling (webcam pose → CSI labels)

The webcam is a teacher used only during data collection. Joint coordinates
extracted from video are joined — on the common time axis established above —
to the CSI window of the same instant, forming (X, Y) training pairs.

```
 webcam mp4 ──► RTMDet-m (person det.) ──► RTMPose-m (COCO-17) ──► BODY-18 conversion
   │ per-frame t_ns                                                │ (neck = shoulder midpoint)
   │              0 persons=no_person · ≥2=multi (discard)         ▼
   │                                       QA gate (random-sample human review, <2%)
   │                                                               │
   ▼                                                               ▼
 t_ns join: frame time = anchor ──► paired with same-instant CSI    Y = PAM (3,18,18)
                                              │                    │  diag = joints (x,y,ĉ)
 9 links (3TX×3RX) × 56 subcarriers × 5 pkts  │                    │
   ──► amplitude tensor X (280,3,3) ──────────┴──► training: f(X) ≈ Y
                                                                   │
                                            inference uses CSI only — no camera
```

Empty-room frames are not discarded — they become presence=0 negative samples
(the loss weighting automatically suppresses the coordinate terms). Because
board oscillators are independent, inter-link phase differences are random
quantities, so **inter-link phase is never used as a feature**; only amplitude
is (intra-link phase shape is an opt-in ablation). Per-board AGC differences
are absorbed by per-link L2 normalization.

## Hardware requirements

- 6 ESP32-S3 dev boards (3 TX + 3 RX, built with ESP-IDF — HT20, 56 subcarriers)
- 6 USB-UART adapters/cables (CH340 etc.) or the boards' built-in USB Serial/JTAG
- 1 USB webcam (teacher label collection only — not needed for inference)
- An MQTT broker (mosquitto, default localhost:1883)

Recommended setup: capture (serial/webcam) on native Windows, training and
real-time inference on WSL2 (ext4) — for timestamp unification and I/O
performance. A single-OS setup works as well.

<p align="center">
  <img src="figures/room-layout.png" alt="Measurement room layout" width="560"><br>
  <em>Measurement room (3.45 × 5.65 m). The TX array (bottom, facing up) and the
  RX + camera cluster (top, facing down) face each other along the long axis;
  the mattress at the center is the fall point.</em>
</p>

## Getting started

```bash
pip install -r requirements.txt

# Local configs — copy the examples and fill in your environment
# (the originals are gitignored)
cp configs/boards.example.yaml configs/boards.yaml   # COM ports · board MACs
cp configs/train.example.yaml  configs/train.yaml    # session h5 paths

# Firmware (ESP-IDF v5.x)
cd firmware/tx && idf.py set-target esp32s3 build flash    # 3 TX boards
cd firmware/rx && idf.py set-target esp32s3 build flash    # 3 RX boards
```

Naming convention: `csi_*` directories (`csi_host/`, `csi_pipe/`, `csi_train/`,
…) are library modules; the sibling top-level scripts (`bridge.py`, `train.py`,
…) are CLI wrappers.

Note: code comments and CLI help strings are written in Korean. "설계 §N" /
"스펙 …" in comments refer to section numbers of an internal design document.

## Attribution & license notices

- **`train/csi_train/model.py`** is an unofficial modified implementation based
  on `models/wisppn_resnet.py` from [geekfeiw/WiSPPN](https://github.com/geekfeiw/WiSPPN).
  Paper: Fei Wang, Stanislav Panev, Ziyi Dai, Jinsong Han, Dong Huang,
  *"Can WiFi Estimate Person Pose?"*, [arXiv:1904.00277](https://arxiv.org/abs/1904.00277) (2019).
  The upstream repository has no license file; copyright of the parts derived
  from it remains with the original authors. This repository uses them with
  attribution.
- The **teacher stage** automatically downloads RTMDet/RTMPose ONNX models from
  download.openmmlab.com ([OpenMMLab mmpose](https://github.com/open-mmlab/mmpose),
  Apache-2.0) on first run. The model files themselves are not included in this
  repository.
- The BODY-18 limb definition in `teacher/csi_teacher/qa.py` is the standard
  OpenPose skeleton topology (factual data).

License of this repository: [MIT](LICENSE). Note that the parts of
`train/csi_train/model.py` derived from the original remain under the original
authors' copyright, independent of this repository's license.
