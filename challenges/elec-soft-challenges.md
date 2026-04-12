# Elec ↔ Soft: Challenges & Deep Dives
**Last updated:** 2026-04-12

---

## Challenge 1: CAN-FD Bus Saturation with 8 Motor Controllers

### Problem
The rover runs 8 CAN-FD nodes on one physical bus:
- 4× AK80-9 drive controllers (IDs 0x01–0x04)
- 4× moteus r4.11 steering controllers (IDs 0x11–0x14)

At 5 Mbit/s CAN-FD, a 64-byte data frame takes approximately 170 µs. If all 8 controllers are queried at 1 kHz, the theoretical minimum bus time is 8 × 170 µs = 1.36 ms per cycle — already above the 1 ms budget.

### Analysis
In practice, CAN-FD uses a faster bit rate for the data phase (up to 5 Mbit/s) but the arbitration phase runs at 1 Mbit/s. AK80-9 and moteus use compact frames, not 64-byte maximum. A typical cyclic command + status frame is 16–24 bytes.

At 16-byte frames, 1 Mbit/s arbitration, 5 Mbit/s data:
- Per frame: ~25 µs data + ~10 µs arbitration = ~35 µs
- 8 controllers, 1 kHz: 8 × 35 µs = 280 µs per cycle
- Bus utilisation: 28% — healthy.

**Recommendation:** Run drive controllers at 1 kHz, steering controllers at 200 Hz (steering dynamics are slower). This reduces bus load to ~55 µs per cycle (5.5% utilisation) with comfortable margin.

### Monitoring
Add a CAN-FD bus error rate monitor in ros2_control. If error rate exceeds 1%, trigger an alert. If it exceeds 5%, trigger the failsafe (see elec-soft.md §3.2). Log all CAN errors with timestamps.

---

## Challenge 2: IMU Vibration Noise from Swerve Drive

### Problem
Swerve drive modules with brushless motors produce vibration in two frequency bands:
- Motor electrical frequency: RPM/60 × pole pairs (typically 50–400 Hz)
- Mechanical resonance: chassis modes (typically 20–80 Hz)

The ICM-20948 accelerometer samples at up to 1.1 kHz. Without filtering, motor vibration aliases into the acceleration signal, corrupting the EKF's dead-reckoning when GPS is unavailable.

### Mitigation
**Hardware:** Mount IMU on 10 Shore A silicone grommets. This provides ~20 dB attenuation above 100 Hz.

**Software (already implemented):** FFT-based notch filter on accelerometer channels. The motor electrical frequency changes with speed. Options:
1. **Static notch at measured peak frequency** — simple, effective if motor RPM range is narrow.
2. **Adaptive notch tracking motor RPM** — subscribe to AK80-9 velocity feedback, compute notch frequency in real time. More complex but handles variable speed.

**Recommendation:** Implement adaptive notch tied to drive motor RPM feedback from CAN. This was already designed in the sensor fusion architecture — confirm it accounts for all 4 drive motors (use the maximum RPM as the notch frequency).

---

## Challenge 3: Power Rail Noise Affecting Sensor ADC

### Problem
The 6S Li-ion drive rail carries high-current pulses from motor controllers (PWM at 20–50 kHz). Even with the compute 4S rail isolated, ground bounce from high-current return paths can couple into sensor ADC references.

Symptoms: IMU readings show periodic spikes correlated with motor activity. GPS position jumps when motors accelerate. Optical flow velocity has quantisation noise.

### Mitigation
1. **Star grounding:** All sensor ground returns connect to a single point on the chassis (chassis ground star). Motor current returns connect to a separate star. The two stars connect at one point only — the battery negative terminal.
2. **Decoupling capacitors:** 100 µF electrolytic + 100 nF ceramic in parallel at the power input of every sensor PCB.
3. **Isolated sensor rail:** Run sensors from a dedicated LDO (e.g., TLV1117 3.3V) fed from the 4S compute rail, not directly from the DCDC converter output.
4. **Keep compute rail DCDC converter physically distant** from IMU (> 100 mm). High-frequency switching noise radiates from the converter.

---

## Challenge 4: Power Sequencing Race Conditions

### Problem
On boot, the Jetson Orin NX takes approximately 15–20 seconds to fully start ROS 2 nodes. During this time, if GPIO 12 defaults HIGH (E-STOP released), the drive rail is live before software has control. An unconfigured ros2_control can send random commands to motor controllers.

### Mitigation
1. **GPIO 12 default:** The E-STOP relay must be wired as **normally open** (NO). GPIO 12 must be pulled LOW by default via a 10 kΩ pull-down resistor. Software must actively drive it HIGH to release the E-STOP.
2. **Boot order:** Relay only closes when software explicitly asserts GPIO 12. Hardware cannot close it prematurely.
3. **Watchdog on GPIO 12:** A hardware monoflop (555 timer or dedicated watchdog IC like MAX6369) retriggers every 500 ms from a software heartbeat. If the heartbeat stops, the monoflop times out and pulls GPIO 12 LOW. This means even a software crash releases the E-STOP.

**Recommended watchdog IC:** MAX6369KA+T — WDI input toggles every 500 ms from a GPIO. Timeout period 1.6 s. Output drives the E-STOP relay gate. No software needed beyond a simple toggle in the watchdog node.

---

## Challenge 5: RC Takeover Latency

### Problem
When the operator presses the RC transmitter emergency takeover, the SBUS signal on GPIO 33 must override ROS 2 within milliseconds. If ROS 2 is currently commanding motors at full speed, there is a window where both ROS 2 commands and RC commands fight for control.

### Mitigation
The RC SBUS signal must be processed **in hardware**, not in a ROS 2 node:

1. SBUS on GPIO 33 connects to the PDU's STM32 microcontroller.
2. PDU STM32 detects valid SBUS signal → immediately disables the ROS 2 CAN-FD path to motor controllers via a hardware multiplexer (e.g., ISO1050 CAN transceiver with enable pin).
3. PDU STM32 decodes SBUS and sends RC commands directly onto CAN-FD bus.
4. Jetson is notified via a separate GPIO (PDU → Jetson interrupt) that RC mode is active. ROS 2 stops publishing to motor controllers.
5. RC-to-motor latency: < 5 ms (SBUS frame is 25 ms, PDU processes in < 1 ms).

This architecture means ROS 2 can crash completely and the operator still has full manual control.

---

## Challenge 6: Dual GPS Antenna Placement

### Problem
Both GPS antennas (M10N primary + SAM-M10Q redundant) must have an unobstructed sky view. On a rover with a 6-DOF arm and drone landing pad, finding two clean antenna locations is hard.

### Constraints
- GPS patch antenna requires a ground plane of at least 100 × 100 mm for optimal reception.
- Antennas must be > 100 mm apart to avoid mutual coupling.
- Arm in any position must not occlude the sky above the antenna.
- Drone rotors must not pass over the antenna.

### Recommendation
1. Mount both GPS antennas on a raised mast (100 mm above chassis top plate), forward of the arm base.
2. Mast location: centreline of chassis, at the front edge — forward of arm swept volume, clear of drone pad.
3. Ground plane: aluminium disc 120 mm diameter under each antenna.
4. Antennas separated by 150 mm laterally on the mast.
5. Run coaxial cable (RG-174 or LMR-100) in a dedicated conduit away from motor power cables.
