# Life Sciences ↔ All Domains Interface
**Boundary:** Science payload integration — volume, power, data, and operation sequences.
**Last updated:** 2026-04-12

---

## 1. Volume & Mass Contract (↔ Mechanical)

Mechanical guarantees this envelope. Life Sciences must fit everything inside it. No exceptions without a mass budget PR.

| Item | Envelope (mm) | Mass allocation | Mount method | Location |
|---|---|---|---|---|
| Science bay total | 220 × 160 × 130 | 4.0 kg total | M4 corner mounts on chassis | Forward-starboard |
| Drill rig | 180 × 60 × 60 | 1.8 kg | Fixed to bay floor, M4 bolts | Bay floor, centre |
| Spectrometer | 100 × 40 × 30 | 0.4 kg | Rail-mounted, removable | Bay wall, port |
| Microscope | 80 × 40 × 40 | 0.3 kg | Downward-pointing, bay floor port | Bay floor, forward |
| Sample containers (×4) | 50 × 50 × 80 each | 0.5 kg total | Bayonet-lock slots | Bay rear |
| pH + moisture + temp probe | 20 × 20 × 150 | 0.2 kg | Extends through bay floor via Teflon gland | Bay floor, aft |
| Wiring + PCB | — | 0.5 kg | CM4 mounted inside bay | Bay ceiling |
| **Bay + contents total** | | **3.7 kg** | | |
| **Margin within allocation** | | **0.3 kg** | | |

**The microscope floor port is a through-hole in the chassis plate.** Mechanical must seal against dust ingress with a retractable shutter or removable plug when not sampling.

---

## 2. Power & Data Contract (↔ Electrical / Software)

All science power is on the **compute 4S rail, gated by GPIO 16 (software-controlled)**. Life Sciences must not request power from the drive rail.

| Instrument | Voltage | Max current | Protocol | Data rate | Connected to |
|---|---|---|---|---|---|
| Drill motor | 12 V regulated (from 4S) | 5 A peak, 2 A continuous | GPIO direction + PWM speed | N/A | RPi CM4 GPIO 4 (enable) + GPIO 5 (PWM) |
| Spectrometer | 5 V USB | 500 mA | USB CDC serial `/dev/ttyUSB0` | ~1 Hz during scan | RPi CM4 |
| Microscope camera | 5 V USB | 500 mA | USB UVC `/dev/video0` | 10 fps @ 1080p during sample | RPi CM4 |
| Moisture sensor | 3.3 V | 20 mA | I2C 0x28, 100 kHz | 1 Hz | RPi CM4 I2C-0 |
| pH sensor | 3.3 V | 20 mA | I2C 0x36, 100 kHz | 1 Hz | RPi CM4 I2C-0 |
| Temperature sensor | 3.3 V | 10 mA | I2C 0x40, 100 kHz | 1 Hz | RPi CM4 I2C-0 |
| CM4 → Jetson bridge | 3.3 V UART | — | UART 921600 | ~10 Hz aggregate | RPi CM4 /dev/ttyAMA0 → Jetson /dev/ttyTHS4 |

---

## 3. Autonomous Operation Sequence (↔ Software)

This is the contract between Life Sciences (who defines the science protocol) and Software (who implements it in `science_mission.py`). Life Sciences must not change this sequence without a PR here and a regression test of the science mission node.

```
Step 1 — Navigate to waypoint
  Rover drives to science GPS waypoint via Nav2.
  Arrival threshold: 1.0 m from target.

Step 2 — Arm probe deployment
  MoveIt2 commands arm to named pose: "science_probe".
  Arm must reach ground contact within 10 s or mission aborts.

Step 3 — Soil condition pre-check
  CM4 reads moisture + pH + temperature.
  If all three sensors NAK: abort, log error, move to next waypoint.
  If any one sensor fails: continue with remaining sensors, flag in CSV.

Step 4 — Drill activation
  CM4 enables drill at 60% PWM for 8 s.
  Overcurrent protection: if current > 5 A for > 200 ms, disable drill immediately,
  retract arm, wait 5 s, retry once. If retry fails, log and move on.

Step 5 — Sensor readings (during and after drill)
  CM4 records moisture, pH, temperature at 1 Hz throughout drill cycle.
  Final reading taken 2 s after drill stops.

Step 6 — Spectrometer scan
  CM4 triggers spectrometer scan. Integration time: 3 s.
  If USB device not present: log "spectrometer_unavailable", skip step.
  If scan returns all zeros: flag as "scan_error", retry once.

Step 7 — Microscope image capture
  CM4 captures 3 frames, saves all three.
  If USB device not present: log "microscope_unavailable", skip step.
  Filename format: YYYYMMDD_HHMMSS_site<N>_frame<M>.jpg

Step 8 — Data logging
  CM4 writes one row to science_results.csv:
  timestamp, GPS_lat, GPS_lon, site_id, moisture_%, pH, temp_C,
  spectrometer_status, microscope_status, drill_cycles, notes

Step 9 — Arm retraction
  MoveIt2 commands arm to "stow" pose.
  Rover can move only after stow confirmed.

Step 10 — Mission continuation
  science_mission.py publishes /science/site_complete (Bool=True).
  Rover proceeds to next waypoint.
```

---

## 4. Science Data Format Contract

`science_results.csv` is the deliverable. Life Sciences defines the schema. Software implements it. Do not change column order without updating both teams.

```
timestamp, site_id, GPS_lat, GPS_lon, moisture_pct, pH, temp_C,
spectro_status, spectro_file, microscope_status, microscope_files,
drill_cycles, drill_overcurrent_events, sensor_fault_flags, notes
```

| Field | Type | Notes |
|---|---|---|
| timestamp | ISO 8601 UTC | From Jetson system clock, chrony-synced to GPS PPS |
| site_id | String | e.g., "ARC2026_site_01" |
| GPS_lat / GPS_lon | Float64 | From primary GPS at time of sampling |
| spectro_status | Enum | "ok" / "unavailable" / "scan_error" |
| microscope_files | String | Comma-separated filenames |
| sensor_fault_flags | Bitmask | Bit 0=moisture, 1=pH, 2=temp, 3=spectrometer, 4=microscope, 5=drill |

---

## 5. Failsafe & Recovery (↔ Software / Electrical)

| Failure | Detection | Action | Data impact |
|---|---|---|---|
| Science bay power not enabled | GPIO 16 LOW when science node starts | Node waits and retries 3× at 2 s intervals, then alerts operator | No data loss |
| Drill overcurrent | CM4 current sense > 5 A for > 200 ms | Disable drill, retract arm, retry once | Log "drill_overcurrent" in CSV |
| Drill stall (motor on, no motion) | Current > 4 A for > 3 s with no soil progress | Disable drill, retract, flag site as "hard_substrate" | Continue to next site |
| I2C sensor NAK | 3 consecutive NAK responses | Mark sensor unavailable, skip readings, continue | Fault flag set in CSV |
| Spectrometer USB disconnect | Device disappears mid-scan | Cancel scan, mark "unavailable", continue | Flag in CSV |
| Microscope USB disconnect | Device disappears | Skip capture, mark "unavailable", continue | Flag in CSV |
| CM4 → Jetson UART loss | No data for > 5 s | Jetson logs "science_bridge_timeout", science mission pauses | Mission waits up to 30 s, then continues without science data |
| CM4 crash / reboot | UART silence | Jetson detects loss, logs event | CM4 auto-reboots (hardware watchdog). Mission resumes after reconnect. |
| Arm fails to reach "science_probe" | MoveIt2 timeout | Abort science step, log "arm_unreachable", move to next waypoint | No data for that site |

---

## 6. Physical Science Bay Access

| Requirement | Owner |
|---|---|
| Bay lid must be openable without tools (thumb screws or quarter-turn fasteners) | Mechanical |
| Sample containers must be extractable without removing any other component | Mechanical |
| Drill shaft must be replaceable in < 5 minutes in field | Mechanical + Life Sciences |
| All USB cables inside bay must be strain-relieved — no bare connectors | Electrical |
| Bay must have at least IP54 dust protection when lid is closed | Mechanical |
