# Electrical ↔ Software Interface
**Boundary:** Where code meets silicon — the data highway.
**Last updated:** 2026-04-12

---

## 1. Pinout & Protocol Master List

This table is the single source of truth. Software cannot write a driver until its row is complete and both teams have signed off (PR merged).

### 1.1 Jetson Orin NX — Primary Sensor Bus

| Bus | Device | Port / CS | Address / ID | Speed | IRQ / PPS | Notes |
|---|---|---|---|---|---|---|
| SPI0 | ICM-20948 IMU (primary) | CS: GPIO 24 | — | 7 MHz | GPIO 25 (DRDY, active LOW) | Hardware interrupt mandatory — polled IMU at 200 Hz will cause Nav2 jitter |
| I2C-0 | BNO085 IMU (redundant) | — | 0x4A | 400 kHz | GPIO 26 (INT, active LOW) | Onboard sensor fusion. Activated if ICM-20948 fails or produces bad data for > 100 ms |
| UART0 | Radiolink M10N GPS (primary) | /dev/ttyTHS0 | — | 115200 | PPS → GPIO 18 | UBX binary protocol. PPS feeds chrony for system time sync. |
| UART1 | u-blox SAM-M10Q GPS (redundant) | /dev/ttyTHS1 | — | 9600 | None | NMEA protocol. Activated on primary GPS fix loss > 3 s. |
| UART2 | MicoAir MTF-01 optical flow + ToF (primary) | /dev/ttyTHS2 | — | 115200 | None | Custom binary frame. ToF height fed to EKF for optical flow de-rotation. |
| SPI1 | PMW3901 optical flow (redundant) | CS: GPIO 27 | — | 2 MHz | GPIO 28 (MOT, motion detect) | Activated if MTF-01 SQUAL = 0 for > 500 ms. No ToF — use BNO085 pitch for de-rotation fallback. |
| CAN-FD | 4× AK80-9 drive motors | — | IDs 0x01–0x04 | 5 Mbit/s | — | TJA1463 transceiver. Cyclic 1 ms CMD, async FB. |
| CAN-FD | 4× moteus r4.11 steer controllers | — | IDs 0x11–0x14 | 5 Mbit/s | — | Same physical bus as drive. CAN-FD arbitration handles both. |
| I2C-1 | INA3221 current/voltage sensor (drive rail) | — | 0x40 | 100 kHz | GPIO 29 (ALERT) | Monitors drive 6S rail + arm rail + drone charge rail simultaneously |
| I2C-1 | INA3221 current/voltage sensor (compute rail) | — | 0x41 | 100 kHz | GPIO 30 (ALERT) | Monitors compute 4S rail + science rail |
| I2C-1 | Battery BMS (cell telemetry) | — | 0x42 | 100 kHz | GPIO 31 (FAULT) | Per-cell voltage + temperature. Read at 1 Hz. |
| USB | RPLIDAR A2M8 | /dev/ttyUSB0 | — | 115200 | — | 2D LiDAR for SLAM + obstacle detection. Must be on dedicated USB port — no hub. |
| UDP | MAVLink 2.0 drone telemetry | 192.168.2.1:14550 | — | Wi-Fi 5 GHz | — | MAVSDK-ROS node on Jetson. Heartbeat timeout = 2 s. |
| UART3 | SiK 915 MHz radio (comms redundancy) | /dev/ttyTHS3 | — | 57600 | — | Fallback comms when Wi-Fi drops. cmd_vel + telemetry only at this bandwidth. |
| GPIO 12 | E-STOP relay (drive + arm power cut) | Output | — | — | Active LOW | Pull LOW to cut drive + arm 6S rail. Independent of ROS 2. |
| GPIO 16 | Science bay power enable | Output | — | — | Active HIGH | Science rail live only when software explicitly enables. |
| GPIO 20 | Drone trickle-charge enable | Output | — | — | Active HIGH | Operator-initiated only. Never auto-enabled. |
| GPIO 33 | RC receiver SBUS input | Input | — | 100 kHz SBUS | — | Hardware override — RC commands bypass ROS 2 entirely when RC signal detected. |

### 1.2 Raspberry Pi 5 (Arm Co-processor)

| Bus | Device | Port | Speed | Notes |
|---|---|---|---|---|
| UART0 | STM32 F407 (arm hardware interface) | /dev/ttyAMA0 | 115200 8N1 | 42-byte CMD frame / 74-byte FB frame. CRC-16/CCITT-FALSE. Watchdog: STM32 reboots if no valid CMD for 500 ms. |
| I2C-0 | (reserved for future arm sensor) | — | 400 kHz | — |

### 1.3 Raspberry Pi CM4 (Science Co-processor)

| Bus | Device | Port | Speed | Notes |
|---|---|---|---|---|
| USB-A | Spectrometer | /dev/ttyUSB0 | USB 2.0 CDC | 1 Hz during scan. 3 s integration time. |
| USB-A | Microscope camera | /dev/video0 | USB 2.0 UVC | 10 fps @ 1080p during sample. |
| I2C-0 | Moisture sensor | — | 0x28, 100 kHz | 1 Hz |
| I2C-0 | pH sensor | — | 0x36, 100 kHz | 1 Hz |
| I2C-0 | Temperature sensor | — | 0x40, 100 kHz | 1 Hz |
| GPIO 4 | Drill motor enable | Output | — | Active HIGH. PWM on GPIO 5 for speed control. |
| UART0 | CM4 → Jetson science data bridge | /dev/ttyAMA0 | 921600 | Aggregated science data upstream to Jetson. |

---

## 2. Power Sequencing Contract

Boot order is a contract. Software must not assume any actuator rail is live before its enable condition is met. Violating this causes uncontrolled motor startup.

| Step | Event | Trigger | Dependency |
|---|---|---|---|
| 1 | Operator closes main switch | Hardware | None |
| 2 | PDU energises compute 4S rail | Hardware auto | Main switch closed |
| 3 | Jetson Orin NX + RPi 5 boot | Hardware | Compute rail live |
| 4 | All ROS 2 core nodes healthy → Software pulls GPIO 12 HIGH | Software | ros2_control lifecycle active |
| 5 | E-STOP relay closes → drive + arm 6S rail live | Hardware (relay) | GPIO 12 HIGH |
| 6 | Software calls `on_configure()` on ros2_control hardware interfaces | Software | Step 5 complete |
| 7 | Software calls `on_activate()` — motors accept commands | Software | Step 6 complete |
| 8 | Operator confirms ready → Software enables GPIO 16 (science rail) | Software (operator-gated) | Step 7 complete |
| 9 | Operator initiates GPIO 20 (drone charge) | Software (operator-gated) | Drone docked confirmed |

**Power-down sequence is the reverse of boot.** Software must call `on_deactivate()` → `on_cleanup()` before cutting rails.

---

## 3. Failsafe & Watchdog Contract

**Every failure must produce a defined, safe behaviour. Undefined = unacceptable.**

### 3.1 Hardware-Level Failsafes (no software dependency)

| Trigger | Action | Implemented by |
|---|---|---|
| E-STOP button pressed | Hardware loop opens immediately. Drive + arm 6S rail cuts in < 10 ms. | Electrical hardware |
| Battery voltage < 20.5 V (6S) | PDU hardware undervoltage cutoff. No software involved. | Electrical hardware — BMS |
| Per-cell voltage < 3.0 V | BMS opens battery contactor. Immediate. | Electrical hardware — BMS |
| Per-cell temperature > 60°C | BMS opens battery contactor. | Electrical hardware — BMS |
| INA3221 current > rail fuse rating | Fuse blows. Physical. | Electrical hardware |
| RC receiver signal detected | RC SBUS signal on GPIO 33 bypasses ROS 2 immediately. Operator has full manual control. | Electrical hardware |

### 3.2 Software Watchdog Failsafes

| Trigger | Condition | Action | Fallback behaviour |
|---|---|---|---|
| ROS 2 heartbeat loss | `/rosout` or `/diagnostics` silent > 2 s | Watchdog node pulls GPIO 12 LOW → E-STOP | Drive + arm power cut |
| CAN-FD bus error rate | Error rate > 5% over 100 ms window | ros2_control `on_error()` → commanded velocity = 0, hold position | Log fault, alert operator |
| Single motor controller offline | CAN heartbeat from one AK80-9 or moteus lost > 200 ms | Disable affected module, switch to **3-wheel limp mode** | Continue mission at reduced speed, alert operator |
| Steering motor failure | moteus fault flag received | Lock steering joint at current angle. Switch to **Ackermann-degraded mode** for that corner. | Mission continues at reduced agility |
| IMU primary failure | ICM-20948 DRDY silent > 100 ms OR gyro saturated for > 50 ms | EKF switches to BNO085. Inflate IMU covariance. Alert operator. | EKF continues with secondary IMU |
| GPS primary fix loss | No valid NavSatFix for > 3 s | navsat_transform switches to secondary GPS. Inflate GPS covariance. | EKF continues with reduced position accuracy |
| Both GPS lost | No valid fix from either for > 5 s | Nav2 switches to local odometry only. Geofence enforced by EKF uncertainty radius. | Autonomous nav degrades — operator advised to take manual control |
| Optical flow primary failure | MTF-01 SQUAL = 0 for > 500 ms | EKF switches to PMW3901. Remove ToF de-rotation (use BNO085 pitch). Inflate flow covariance. | EKF continues with secondary flow |
| Both optical flow sensors failed | PMW3901 also failing or SQUAL = 0 | EKF disables optical flow measurement. Wheel odometry + GPS only. | Position accuracy degrades on slippery terrain |
| LiDAR failure | RPLIDAR scan silent > 1 s | Nav2 obstacle layer switches to RGBD depth-projected scan. Alert operator. | Navigation continues with reduced obstacle range |
| RGBD + LiDAR both down | Both obstacle sources dead | Nav2 drops to GPS-only navigation. Max speed capped at 0.3 m/s. | Operator advised to supervise closely |
| Arm UART loss | No valid FB frame from STM32 for > 500 ms | RPi 5 sends E-STOP frame. MoveIt2 cancels all goals. Arm holds last position. | No arm motion until UART restored and `on_configure()` reissued |
| Arm joint overcurrent | STM32 reports joint current > limit for > 100 ms | STM32 disables that joint. RPi 5 notifies MoveIt2. MoveIt2 stops all arm motion. | Arm stowed if possible. Fault logged. |
| Science sensor disconnect | USB device disappears or I2C NAK for > 3 retries | CM4 logs error, marks sensor as unavailable, continues with remaining sensors | Science mission continues with degraded data — logged in results CSV |
| Drill overcurrent | GPIO current sense > 5A for > 200 ms | CM4 disables drill GPIO immediately. Retracts arm. Logs event. | Science mission pauses. Operator can retry. |
| Drone comms loss | MAVLink heartbeat timeout > 2 s | MAVSDK issues RTL command over SiK 915 MHz backup link | Drone returns autonomously |
| Drone battery < 20% | MAVLink battery status | MAVSDK issues RTL or land at current position | Drone lands. Mission data preserved. |
| Wi-Fi comms loss | ROS 2 DDS heartbeat timeout > 1 s | Switch operator station to SiK 915 MHz. Send safe-stop to rover. | Rover stops. Operator resumes on backup comms. |
| Compute rail undervoltage | INA3221 ALERT on GPIO 30 | Watchdog node shuts down non-critical processes. Preserve Nav2 + E-STOP node. | System runs in minimal safe mode |

### 3.3 Drive Degradation Modes

| Mode | Condition | Capability lost | Capability retained |
|---|---|---|---|
| Normal (4WS + 4WD) | All modules healthy | — | Full swerve |
| 3-wheel limp | One drive motor offline | ~25% traction reduction, turning limited | Can still navigate to safe stop or base |
| Ackermann-degraded | One steering motor locked | Crab/holonomic motion limited | Differential-like steering still possible |
| 2-wheel differential | Two drive motors on same side offline | Swerve and crab impossible | Tank-turn navigation only |
| Safe stop | Any two or more faults | All autonomous motion | E-STOP, operator takes over RC |

---

## 4. Comms Redundancy Switching

| Layer | Technology | Bandwidth | Auto-switch trigger |
|---|---|---|---|
| Primary | 5 GHz Wi-Fi | High — full ROS DDS | Default |
| Secondary | SiK 915 MHz `/dev/ttyTHS3` | Low — cmd_vel + diagnostics only | Wi-Fi heartbeat lost > 1 s |
| Tertiary | RC SBUS GPIO 33 | Control only — no data | Operator manual. Always available. |

Layer switching is automatic for Primary → Secondary. Secondary → Tertiary is always operator-initiated.
