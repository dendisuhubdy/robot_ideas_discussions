# How does SLAM work in robotics?

- **Date:** 2026-06-18
- **Topic:** Robotics / Perception / State Estimation

## Question

How does SLAM work in robotics?

## Answer

**SLAM** = *Simultaneous Localization and Mapping*. It solves a chicken-and-egg
problem: to build a map you need to know where you are, but to know where you
are you need a map. SLAM estimates both at once from sensor data as the robot
moves.

### The core problem

The robot maintains two coupled unknowns:

- **Pose** — its own position and orientation over time (the trajectory).
- **Map** — the locations of landmarks/features or the structure of the world.

Formally, given a sequence of **controls** (odometry/IMU) and **observations**
(camera, LiDAR), estimate the posterior over poses and map:

```
p(x_1:t , m | z_1:t , u_1:t)
```

where `x` = poses, `m` = map, `z` = observations, `u` = motion commands.

### The two building blocks

1. **Motion model** — predicts the next pose from the previous pose plus a
   control input (wheel odometry, IMU integration). This *accumulates drift*:
   small errors compound over time.
2. **Observation model** — relates what the sensor *should* see from a given
   pose and map to what it *actually* sees. The mismatch (the residual) is what
   gets corrected.

### The processing pipeline

Most modern SLAM systems share this structure:

1. **Front end (data association / odometry)**
   - Extract features (visual keypoints, LiDAR edges/planes) or use dense
     methods (direct photometric alignment).
   - Match current observations to previous ones → estimate incremental motion.
   - This is where most failures originate (wrong matches = corrupted map).

2. **Back end (optimization)**
   - Fuse all constraints into a globally consistent estimate.
   - Two dominant formulations (below).

3. **Loop closure**
   - Recognize a *previously visited* place ("I've been here before").
   - Adds a constraint that ties the trajectory back on itself, canceling
     accumulated drift. This is what turns a drifting odometry trail into a
     consistent map.

4. **Map management** — keyframe selection, marginalizing old states,
   maintaining the map representation (point cloud, voxel grid, mesh, etc.).

### Two main mathematical approaches

| Approach | Idea | Examples |
|---|---|---|
| **Filtering** | Recursively update a single best estimate + covariance each step | EKF-SLAM, FastSLAM (particle filter) |
| **Graph / smoothing** | Build a factor graph of all poses + constraints, solve by nonlinear least squares | g2o, GTSAM, Ceres — used by ORB-SLAM, Cartographer |

Graph-based **pose-graph optimization** dominates today. Nodes = poses (and
sometimes landmarks); edges = constraints from odometry, observations, and loop
closures. Solving = minimizing total constraint error (bundle adjustment in the
visual case).

### Sensor variants

- **Visual SLAM (vSLAM)** — cameras. *ORB-SLAM3* (feature-based), *DSO/LSD-SLAM*
  (direct), *SVO* (semi-direct).
- **Visual-Inertial (VIO/VI-SLAM)** — camera + IMU; far more robust to fast
  motion and scale ambiguity. *VINS-Mono*, *OKVIS*, *Kimera*.
- **LiDAR SLAM** — 2D/3D range scans, very accurate geometry. *Cartographer*,
  *LOAM/LeGO-LOAM*, *LIO-SAM* (LiDAR-inertial).
- **RGB-D SLAM** — depth cameras indoors. *KinectFusion*, *ElasticFusion*.

### Why it's hard in practice

- **Drift** accumulates without loop closure.
- **Data association** errors are catastrophic and hard to undo.
- **Dynamic environments** (moving people/cars) violate the static-world
  assumption.
- **Scale** is unobservable with a single camera (needs IMU, stereo, or known
  size).
- **Computational budget** — must run in real time on the robot.

### Where the field is heading

- **Learned front ends** — deep feature matching (SuperPoint/SuperGlue),
  learned depth and odometry.
- **Neural/implicit maps** — NeRF- and Gaussian-Splatting-based SLAM
  (e.g. NICE-SLAM, SplaTAM) that build photorealistic, dense maps.
- **Semantic SLAM** — maps that contain object identities, not just geometry,
  bridging toward scene understanding and manipulation.

### Mental model in one line

> SLAM is continuously *predicting* where you moved (motion model), *correcting*
> that with what you see (observation model), and *snapping the whole trajectory
> straight* whenever you recognize a place you've been before (loop closure).
