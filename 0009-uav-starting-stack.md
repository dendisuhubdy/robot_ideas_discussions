# Suggested starting stack for a UAV

- **Date:** 2026-06-18
- **Topic:** Robotics / UAV / System Architecture

## Question

Suggest a starting stack for a UAV.

## Answer

This is an opinionated, buildable baseline for a small autonomous UAV doing onboard perception + GPS-denied navigation. The theme: **one capable compute module, a tight sensor set, quantized CNNs, and a clean split between the flight controller (hard real-time) and the companion computer (perception/autonomy).**

### The two-brain architecture (do this first)

UAVs split into two compute domains — get this boundary right and everything else is easier:

1. **Flight controller (FC)** — hard real-time, runs the autopilot. Handles stabilization, motor mixing, IMU loop, failsafes. **PX4** or **ArduPilot** on a **Pixhawk**-class board. You do *not* run ML here.
2. **Companion computer** — Linux, runs perception, SLAM, planning, and the model-ops daemon. Talks to the FC over **MAVLink** (via MAVSDK or microRTPS/uXRCE-DDS).

Everything below lives on the companion computer except the FC line.

### Compute

- **Start:** Jetson **Orin Nano** (INT8, ~tens of TOPS, ~7–15 W) — the price/power sweet spot for a small UAV.
- **Scale up:** Jetson **Orin NX** or **Thor** if you need FP8/FP4, bigger models, or on-device VLM/LLM tasking.
- Pin `nvpmodel` to a fixed power mode so thermal/battery stay predictable.

### Sensors

- **Stereo or depth camera** — e.g. global-shutter stereo (global shutter matters for fast flight). Primary obstacle + depth source.
- **Downward optical-flow + rangefinder (ToF/lidar-lite)** — altitude hold and velocity estimation indoors / GPS-denied.
- **IMU** (on the FC) — fused for VIO.
- **GPS/GNSS** — outdoor global position when available.
- Optional: thermal or extra mono cameras per mission.

### Perception models (all quantized INT8 → TensorRT)

A lean real-time stack (ties to `0008`):

- **Detection:** YOLOv11-n/s — obstacles, targets, landing markers.
- **Depth:** stereo matching (HW/CUDA) or **Depth Anything-S** if monocular.
- **Tracking:** ByteTrack on top of the detector (cheap — reuses detector output).
- **Localization front end:** **SuperPoint + LightGlue** feeding a VIO/SLAM back end.
- Keep total perception latency budget explicit (e.g. <30–50 ms/frame) so the control loop stays fed.

### State estimation / SLAM (GPS-denied)

- **VIO:** **VINS-Fusion** or the PX4-native EKF2 fused with a vision pose, or **OpenVINS**.
- **Map/SLAM (if needed):** ORB-SLAM3 or a LiDAR-inertial stack if you add LiDAR.
- Feed the resulting pose to the FC as an external vision estimate (MAVLink `VISION_POSITION_ESTIMATE`).

### Autonomy / planning

- **Local obstacle avoidance + planning:** a reactive layer (e.g. an Ego-Planner / mapless local planner) consuming the depth + detections.
- **Mission logic:** behavior layer on the companion computer issuing MAVLink offboard setpoints.
- **Global nav:** waypoints when GPS is available; visual/relative when not.

### Middleware / software

- **ROS 2** (Humble/Jazzy) as the glue — nodes for camera, perception, SLAM, planner.
- **MAVROS** or **microXRCE-DDS** bridge to PX4.
- Containerize perception nodes; keep the FC firmware separate.

### Model-ops / runtime (from `0006`)

- Long-lived **inference daemon** (systemd) that loads the TensorRT engine once, keeps it warm, hot-swaps on update, with an LKG rollback slot.
- **Pull-based OTA** for models on the ground; signed + hashed artifacts; encrypt at rest + secure boot.

### Minimal viable v0 (if you want to start tiny)

> Pixhawk + PX4, Jetson Orin Nano, one stereo camera, ROS 2, YOLOv11-n (INT8/TensorRT) for detection, EKF2/VIO for pose, MAVSDK offboard control. Get *that* flying and perceiving before adding LiDAR, SLAM maps, or VLM tasking.

### Decision points worth confirming

- **Indoor/GPS-denied vs outdoor?** Drives how much you invest in VIO/SLAM vs GNSS.
- **Payload/size class?** Sets the compute (Orin Nano vs Thor) and sensor weight budget.
- **Do you need natural-language tasking / onboard reasoning?** Only then does the heavier VLM/LLM tier (and Thor) earn its place.
