# Interface Control Document — Master Index
**Team:** [Team Name]
**Competition:** Anatolian Rover Challenge (ARC) — primary. ERC / URC compatible.
**Last updated:** 2026-04-12
**Status:** DRAFT — all values marked TBD must be filled before first field test

---

## How to use this document

The ICD defines the **boundaries** between sub-teams. These are not suggestions.
Any change to a value in this document requires a PR reviewed by both affected sub-teams.
No verbal agreements. No WhatsApp changes. PR or it didn't happen.

**Boundary files:**
| File | Interface |
|---|---|
| [mech-elec.md](mech-elec.md) | Mechanical ↔ Electrical |
| [elec-soft.md](elec-soft.md) | Electrical ↔ Software |
| [soft-mech.md](soft-mech.md) | Software ↔ Mechanical |
| [lifesci-all.md](lifesci-all.md) | Life Sciences ↔ Mechanical / Electrical / Software |
| [drone-rover.md](drone-rover.md) | Drone ↔ Rover (all domains) |

**Deep-dive challenge docs:**
| File | Contents |
|---|---|
| [challenges/mech-elec-challenges.md](challenges/mech-elec-challenges.md) | Steering joint cable routing, thermal edge cases |
| [challenges/elec-soft-challenges.md](challenges/elec-soft-challenges.md) | CAN-FD saturation, EMI, power sequencing risks |
| [challenges/soft-mech-challenges.md](challenges/soft-mech-challenges.md) | Swerve kinematics, arm + Nav2 coordination |
| [challenges/lifesci-challenges.md](challenges/lifesci-challenges.md) | Sensor integration, drill sequencing, data pipeline |
| [challenges/drone-challenges.md](challenges/drone-challenges.md) | MAVLink-ROS 2 bridge, RF coexistence, failsafe modes |

---

## Iron Triangle — The Three Laws

These numbers are absolute. A sub-team that needs more must take it from another sub-team's allocation, approved by team lead.

### Law 1: Mass Budget
**Hard ceiling: 50 kg. Nominal build target: 45 kg. Enforced margin: 5 kg (untouchable without team lead vote).**

| Sub-team | Allocation | Key constraints |
|---|---|---|
| Mechanical (chassis + 4 swerve modules) | 20 kg | CF main plate + aluminium swerve brackets. Each module ≤ 1.6 kg. |
| Electrical (batteries + PDU + wiring + motor controllers) | 10 kg | 21700 Li-ion cells mandatory — LiPo banned (field safety). |
| Arm (6-DOF + gripper + mount bracket) | 7 kg | CF links where possible. |
| Science payload (drill + spectrometer + microscope + sensors + containers) | 4 kg | Drill rig is mass anchor — all other science items compete for remaining 2.2 kg. |
| Compute + comms + cameras + LiDAR | 2 kg | Orin NX module = 45 g. Total enclosures + cables stay under budget. |
| Drone (airframe + battery, docked on rover) | 2 kg | 5" hex. Must stay under 2 kg AUW including battery. |
| **Enforced margin** | **5 kg** | **No team touches this. Team lead vote required to release any margin.** |
| **Total** | **50 kg** | |

### Law 2: Power Budget
**System voltage: 6S Li-ion (22.2 V nominal). Drive + arm rail: 2× 6S 20 Ah in parallel. Compute rail: 1× 4S 10 Ah isolated.**

| Subsystem | Continuous (W) | Peak (W) | Rail | Owner |
|---|---|---|---|---|
| Drive motors — 4× AK80-9 | 160 | 480 | Drive 6S | Electrical |
| Steering motors — 4× GB36 + moteus | 30 | 80 | Drive 6S | Electrical |
| Arm actuators (6 joints + gripper) | 40 | 150 | Drive 6S | Electrical |
| Jetson Orin NX 8GB | 15 | 25 | Compute 4S | Software |
| RPi 5 + STM32 F407 (arm co-proc) | 10 | 12 | Compute 4S | Software |
| RPi CM4 (science) | 5 | 8 | Compute 4S | Life Sciences |
| RPLIDAR A2M8 | 3 | 5 | Compute 4S | Software |
| Cameras (×4) + depth sensor | 8 | 12 | Compute 4S | Software |
| IMU (ICM-20948 + BNO085) | 1 | 1 | Compute 4S | Software |
| GPS (M10N primary + backup) | 1 | 2 | Compute 4S | Software |
| Science payload (drill + sensors) | 20 | 45 | Compute 4S | Life Sciences |
| Comms (SiK 915 MHz + Wi-Fi AP) | 10 | 15 | Compute 4S | Software |
| Drone trickle-charge (docked) | 20 | 60 | Drive 6S | Drone |
| **Margin** | | **~100 W** | | |
| **Total peak** | | **~895 W** | | |

### Law 3: Compute Budget
**Platform: NVIDIA Jetson Orin NX 8GB — 8-core ARM Cortex-A78AE + 1024-core Ampere GPU.**

| Task | CPU % | GPU % | Notes |
|---|---|---|---|
| Nav2 + dual EKF state estimation | 25 | 0 | Must never be starved — highest priority |
| Swerve kinematics solver (100 Hz) | 10 | 0 | Real-time, runs in isolated thread |
| Vision: ArUco / gate / YOLO + drone feed | 15 | 55 | GPU-bound — allocate before everything else |
| MoveIt2 arm planning | 15 | 5 | Bursty — runs only during manipulation |
| Science data acquisition + logging | 5 | 0 | On CM4 primarily; Jetson only for aggregation |
| Drone MAVLink bridge + map ingestion | 5 | 0 | UDP processing, low CPU |
| Sensor drivers (IMU, LiDAR, GPS, flow) | 5 | 0 | Interrupt-driven — must not be deprioritised |
| **Margin** | **20** | **40** | |

---

## Recommended Hardware Stack

### Compute

| Role | Board | Rationale |
|---|---|---|
| Main computer | NVIDIA Jetson Orin NX 8GB | 1024-core Ampere GPU handles YOLO + Nav2 + drone map simultaneously. Jetson Nano will saturate. |
| Arm co-processor | Raspberry Pi 5 (8 GB) | Dedicated UART ↔ STM32 F407 arm protocol. Isolated from nav jitter. |
| Science payload processor | Raspberry Pi CM4 (4 GB) | USB spectrometer + microscope + I2C sensors — isolated compute, no Nav2 competition. |
| Drone companion computer | Raspberry Pi Zero 2W | Runs ArUco detection + MAVLink ↔ ROS 2 bridge onboard drone. |

### Drive System (4WS + 4WD Swerve)

| Component | Part | Qty | Notes |
|---|---|---|---|
| Drive motor + integrated controller | CubeMars AK80-9 | 4 | 18 Nm peak, integrated CAN controller + absolute encoder. One CAN cable per module. |
| Steering motor | T-Motor GB36-2 BLDC | 4 | ~2 Nm steer torque. Compact. |
| Steering controller | MJBots moteus r4.11 | 4 | 40×40 mm CAN-FD. Pairs with GB36 + AS5600. |
| Steering absolute encoder | AS5600 magnetic | 4 | Survives dust and vibration. No optical encoder on steering axis. |
| CAN-FD transceiver | TJA1463 | 1 | On Jetson carrier board. Handles 8 nodes on one bus. |
| Bus | CAN-FD @ 5 Mbit/s | — | Regular CAN 2.0 saturates with 8 controllers at high update rate. |

### Sensors — Primary + Redundant

| Sensor | Primary | Redundant / Backup | Failover behaviour |
|---|---|---|---|
| IMU | ICM-20948 (SPI, 200 Hz) | BNO085 (I2C, 100 Hz, onboard fusion) | EKF switches input source; covariance inflated during switch |
| GPS | Radiolink M10N (UART + PPS) | u-blox SAM-M10Q (UART, second port) | navsat_transform uses whichever has fix; Mahalanobis gate rejects outliers |
| Optical flow | MicoAir MTF-01 (UART) | PMW3901 (SPI, 40 Hz) | EKF detects SQUAL=0 on primary, switches to secondary; covariance inflated |
| 2D obstacle mapping | RPLIDAR A2M8 (USB/UART) | RGBD depth image projected to scan | Nav2 obstacle layer has two sources; if LiDAR dies, RGBD scan takes over |
| Depth / 3D obstacles | RGBD camera (USB) | LiDAR point cloud (downsampled) | costmap falls back gracefully |

### Power

| Component | Part | Notes |
|---|---|---|
| Drive battery | 2× 6S 21700 Li-ion 20 Ah (parallel) | Samsung 50E cells, spot-welded. 40 Ah total at 22.2 V. |
| Compute battery | 1× 4S 21700 Li-ion 10 Ah | Isolated rail. Protects Jetson from motor noise. |
| BMS | Active BMS with cell-level monitoring + CAN reporting | Per-cell voltage and temperature reported to Jetson over I2C |
| PDU | Custom STM32-based PDU with INA3221 per rail | Current + voltage sensing, per-rail software enable |
| Per-rail fusing | ATO blade fuses per subsystem | Hardware protection independent of software |

### Communications — Three-Layer Redundancy

| Layer | Technology | Range | Bandwidth | Failover trigger |
|---|---|---|---|---|
| Primary | 5 GHz Wi-Fi (TP-Link directional AP) | ~300 m LOS | High (video + map + ROS topics) | Default |
| Secondary | SiK 915 MHz radio | ~1 km LOS | Low (telemetry + cmd_vel only) | Wi-Fi heartbeat lost > 1 s |
| Tertiary | RC transmitter (FrSky R9M) | ~2 km LOS | Control only (no data) | Operator manual override always available |

### Drone

| Component | Part |
|---|---|
| Flight controller | Cube Orange+ (Pixhawk standard) |
| Telemetry | SiK 915 MHz (shared with rover secondary comms layer) |
| Video | H.264 UDP stream, 5 GHz Wi-Fi |
| Frame | 5" hexacopter (6-rotor redundancy for payload delivery) |
| Companion computer | Raspberry Pi Zero 2W |

---

## Failsafe Philosophy

**Every subsystem must have a defined behaviour for every failure mode. "Hang" is never an acceptable state.**

The failsafe hierarchy is:
1. **Degrade gracefully** — reduce capability, continue mission
2. **Safe stop** — halt motion, hold position, alert operator
3. **E-STOP** — cut power to actuators immediately

No subsystem may block another subsystem's failsafe path.
