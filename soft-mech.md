# Software ↔ Mechanical Interface
**Boundary:** The kinematic contract — physical reality baked into code.
**Last updated:** 2026-04-12

Software cannot control what it doesn't know. Every number in this document must be measured from the physical rover after build and updated here before any autonomous testing begins.

---

## 1. Swerve Kinematic Contract

These values are baked into `rover_with_arm.urdf.xacro`, the swerve controller, and the Nav2 footprint. Mechanical must not change wheel diameter, module positions, or steering axis location without a PR to this file and the URDF.

### 1.1 Wheel & Module Parameters

| Parameter | Value | Status | Measured by |
|---|---|---|---|
| Wheel radius | TBD (m) | Awaiting build | Mechanical — measure loaded radius under 50 kg |
| Wheel width | TBD (m) | Awaiting build | Mechanical |
| Tyre contact patch width | TBD (m) | Awaiting build | Mechanical — critical for Nav2 footprint |
| Drive direction convention | Positive encoder = forward | Fixed | Software |
| Steer zero definition | Wheel pointing +X chassis = 0° | Fixed | Software + Electrical (set at AS5600 assembly) |
| Steer positive direction | Counter-clockwise viewed from above = positive | Fixed | Software |

### 1.2 Module Offsets from Chassis Centre of Mass

All values in metres, in chassis frame. X = forward, Y = left, Z = up. Measured from chassis CoM, not geometric centre.

| Module | X offset (m) | Y offset (m) | Status |
|---|---|---|---|
| Front Left (FL) | TBD | TBD | Awaiting build |
| Front Right (FR) | TBD | TBD | Awaiting build |
| Rear Left (RL) | TBD | TBD | Awaiting build |
| Rear Right (RR) | TBD | TBD | Awaiting build |

**Deadline:** These values must be in this document before the swerve controller is tuned. Software will use placeholder values of ±0.3 m for simulation only.

### 1.3 Steering Limits

| Parameter | Value | Owner |
|---|---|---|
| Physical hard stop | ±195° | Mechanical — set by physical pin or housing |
| Software soft limit | ±180° | Software — 15° safety buffer before hard stop |
| Maximum steering rate | 360°/s | Software — enforced in swerve kinematics solver |
| Continuous rotation | Not supported — cable routing forbids it | Mechanical constraint (coiled cable) |

---

## 2. Arm Physical Limits (URDF Contract)

Mechanical defines the limits. Software bakes them into the URDF joint limits and MoveIt2 constraints. Breaching a soft limit triggers a deceleration. Breaching a hard stop is a mechanical failure.

| Joint | Soft limit (°) | Hard stop (°) | Max velocity (°/s) | Max torque (Nm) |
|---|---|---|---|---|
| Joint_1 (base rotate) | ±170 | ±180 | 60 | TBD by Mech |
| Joint_2 (shoulder) | −10 to +170 | −15 to +175 | 45 | TBD by Mech |
| Joint_3 (elbow) | ±145 | ±150 | 60 | TBD by Mech |
| Joint_4 (wrist roll) | ±170 | ±180 | 90 | TBD by Mech |
| Joint_5 (wrist pitch) | ±120 | ±130 | 90 | TBD by Mech |
| Joint_6 (wrist yaw) | ±170 | ±180 | 120 | TBD by Mech |
| Finger_1 / Finger_2 | 0–45 | 0–50 | 30 | TBD by Mech |

### 2.1 Named Arm Poses (MoveIt2 contract)

These named states are defined in the SRDF and must be physically reachable without joint limit violations.

| Pose name | Purpose | Constraint |
|---|---|---|
| `stow` | Safe for driving — arm fully retracted, inside rover footprint | Must not extend beyond chassis boundary in any direction |
| `deploy_ready` | Arm positioned for picking — pre-grasp height above ground | Elbow and wrist clear of chassis — no self-collision |
| `science_probe` | Arm lowered for soil sampling | End-effector contacts ground at ±50 mm from arm base X-axis |
| `camera_view` | Arm holds camera at elevated position for aerial-like view | End-effector ≥ 400 mm above chassis top plate |

---

## 3. Mass & Inertia Properties

Software needs these values to tune Nav2 MPPI controller, MoveIt2 collision model, and EKF dynamics model. Mechanical provides from CAD. Field-measured values override CAD values.

| Property | Value | Source | Deadline |
|---|---|---|---|
| Total rover mass (fully loaded) | TBD (kg) | Mechanical — weigh after assembly | Before Nav2 MPPI tuning |
| Chassis CoM — X from geometric centre | TBD (m) | Mechanical — CAD or tipping test | Before MPPI tuning |
| Chassis CoM — Y from geometric centre | TBD (m) | Mechanical | Before MPPI tuning |
| Chassis CoM — Z from ground | TBD (m) | Mechanical | Before tip-over risk assessment |
| Chassis yaw moment of inertia (Izz) | TBD (kg⋅m²) | Mechanical — CAD export | Before MPPI tuning |
| Arm link masses (per link) | TBD | Mechanical — CAD export or weigh each link | Before MoveIt2 real-hardware test |
| Arm link inertia tensors (per link) | TBD | Mechanical — CAD export | Before MoveIt2 real-hardware test |
| Science payload CoM shift (loaded vs empty) | TBD (m) | Mechanical — weigh full science bay | Before field science test |
| Swerve module mass (per module) | TBD (kg) | Mechanical | Before swerve controller gain tuning |

---

## 4. Nav2 Footprint Contract

The Nav2 footprint polygon is a living value — it changes depending on arm state. Software updates it dynamically. Mechanical must provide the physical dimensions that define each state.

| State | Footprint polygon | Notes |
|---|---|---|
| Driving (arm stowed) | TBD — rectangular hull of chassis + stowed arm | Measured from top view, add 30 mm safety margin |
| Arm deployed — picking | TBD — extends forward by arm reach | Updated dynamically via `/global_costmap/footprint` |
| Science probe lowered | TBD — extends forward, reduced lateral | |
| Drone docked | Same as arm stowed — drone folds within chassis hull | Drone rotor blades folded for transport |

---

## 5. Terrain & Tipping Limits

Software uses these to trigger slope-based speed limiting and E-STOP. Mechanical must validate through tipping tests with full mass loaded.

| Condition | Threshold | Software action |
|---|---|---|
| IMU roll > 20° | Warning zone | Reduce max speed to 50% |
| IMU roll > 30° | Danger zone | Reduce max speed to 20%, alert operator |
| IMU roll > 38° | Tip risk | E-STOP. Arm commanded to stow immediately. |
| IMU pitch > 25° | Warning zone | Reduce max speed to 50% |
| IMU pitch > 35° | Danger zone | Reduce max speed to 20%, alert operator |
| IMU pitch > 42° | Tip risk | E-STOP. Arm commanded to stow immediately. |
| Arm deployed + pitch > 20° | Combined risk | Force arm to stow. Resume when pitch < 15°. |

**Tip angle limits are TBD until Mechanical performs loaded tipping test.** Values above are conservative placeholders. Mechanical must provide validated tip angles before field testing.

---

## 6. Arm Mounting Offset (URDF fixed joint)

The arm is mounted on the rover chassis. This fixed joint defines how the two kinematic trees connect. These values must be measured precisely after physical mounting.

| Parameter | Value | Notes |
|---|---|---|
| Arm base X offset from chassis origin | TBD (m) | Positive = forward |
| Arm base Y offset | TBD (m) | Positive = left |
| Arm base Z offset | TBD (m) | Positive = up |
| Arm base roll | TBD (°) | Must be 0° unless arm is tilted at mount |
| Arm base pitch | TBD (°) | Must be 0° for standard mount |
| Arm base yaw | TBD (°) | 0° = arm points forward |

**Deadline:** These values must be measured and entered before first combined rover + arm simulation run.
