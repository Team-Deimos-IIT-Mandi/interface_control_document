# Drone ↔ Rover Interface
**Boundary:** Aerial scout + payload delivery + marker detection integrated with rover stack.
**Last updated:** 2026-04-12

---

## 1. Physical Docking (↔ Mechanical)

| Parameter | Spec | Owner |
|---|---|---|
| Landing pad dimensions | 300 × 300 mm flat surface | Mechanical |
| Landing pad location | Aft of arm, centred on rover width, top plate | Mechanical |
| Pad surface | Carbon fibre weave — no raised edges within 250 mm centre zone | Mechanical |
| Drone foot spread | ≤ 220 mm between leg tips (drone team must match this) | Drone team |
| Clearance cone above pad | 400 mm height — no antennas, brackets, or cables inside | Mechanical |
| Retention method | Magnetic retention (neodymium N52, 4× corners) + charge pogo alignment | Mechanical |
| Charge connector | XT30 spring-loaded pogo pins, 2-pin, centred on pad | Electrical |
| Pogo alignment tolerance | ±5 mm lateral, ±3° rotation | Mechanical (guide rails) + Drone (leg precision) |
| Rotor fold clearance | All rotors folded must clear within 280 × 280 mm envelope | Drone team |
| Vibration isolation | 4× silicone dampeners between pad and chassis (10 Shore A) | Mechanical — protects rover sensors during drone motor test |

---

## 2. Power Sharing (↔ Electrical / Software)

| Mode | Source | Voltage | Max current | Enabled by |
|---|---|---|---|---|
| Trickle charge (docked, rover active) | Drive 6S rail via DCDC converter | Drone battery voltage ±0.5 V | 3 A | GPIO 20 — software-gated, operator-initiated only |
| Fast charge (docked, post-mission) | External charger via pad XT30 | Drone battery voltage | 10 A | Operator manual switch on PDU — NOT software-controlled |
| Drone independent flight | Drone internal 4S 3000 mAh LiPo | — | — | Drone FC autonomous |

**Trickle charge is never auto-enabled.** An operator must confirm drone is docked and stable before software enables GPIO 20. Auto-enable is banned — a partially docked drone being charged is a fire risk.

---

## 3. Communication Architecture (↔ Software)

### 3.1 Channel Map

| Channel | Protocol | Transport | Direction | Data | Port / Topic |
|---|---|---|---|---|---|
| Primary telemetry | MAVLink 2.0 | UDP, 5 GHz Wi-Fi | Drone → Rover | Position, altitude, battery, mode, GPS, attitude | 192.168.2.1:14550 |
| Primary command | MAVLink 2.0 | UDP, 5 GHz Wi-Fi | Rover → Drone | Waypoints, takeoff, land, RTL, mode change | 192.168.2.1:14550 |
| Redundant telemetry + command | MAVLink 2.0 | SiK 915 MHz UART | Bidirectional | Telemetry + minimal command (no video) | /dev/ttyTHS3 on rover |
| FPV video | H.264 UDP stream | 5 GHz Wi-Fi | Drone → Rover | Raw camera feed | port 5600 |
| Scout map tiles | GeoTIFF, chunked UDP | 5 GHz Wi-Fi | Drone → Rover | Aerial orthophoto for Nav2 costmap layer | port 5601 |
| ArUco detection result | ROS 2 topic | Wi-Fi ROS DDS | Drone RPi → Rover Jetson | Detected marker pose in world frame | /drone/aruco_pose (PoseStamped) |
| Drone odometry | ROS 2 topic | Wi-Fi ROS DDS | Drone RPi → Rover Jetson | Drone position in world frame | /drone/odometry (Odometry) |

### 3.2 ROS 2 Bridge (Drone Side)

The drone carries a **Raspberry Pi Zero 2W** running:
- `aruco_detection.py` (same node as rover ground version, different camera topic `/drone/camera/image`)
- `mavros` or `MAVSDK-ROS` bridge: publishes MAVLink telemetry as ROS 2 topics
- Connects to rover Jetson ROS 2 DDS domain over Wi-Fi (same ROS_DOMAIN_ID)

### 3.3 ROS 2 Bridge (Rover Side)

Jetson runs:
- `mavros` node subscribing to drone MAVLink over UDP
- `drone_map_ingestor` node: receives GeoTIFF tiles, projects to Nav2 costmap layer
- `drone_aruco_relay` node: subscribes to `/drone/aruco_pose`, injects into rover's ArUco pipeline

---

## 4. MAVLink Command Interface

Commands issued by rover to drone are structured MAVLink missions. Rover software must not issue raw RC override commands — use mission items and `SET_MODE` only.

| Command | MAVLink message | Condition for issue |
|---|---|---|
| Takeoff | `MAV_CMD_NAV_TAKEOFF` | Operator confirms, drone docked check passes |
| Fly to waypoint | `MAV_CMD_NAV_WAYPOINT` | Drone airborne + GPS fix |
| Begin ArUco search pattern | `MAV_CMD_NAV_LOITER_UNLIM` at search altitude | Rover has reached GPS waypoint and started ground search |
| Drop payload | `MAV_CMD_DO_SET_SERVO` (trigger release mechanism) | Drone at designated drop point ± 2 m |
| Return to rover | `MAV_CMD_NAV_RETURN_TO_LAUNCH` | Any time — also issued automatically on comms loss |
| Land at rover pad | `MAV_CMD_NAV_LAND` at rover GPS position | Rover stationary, operator confirms |
| Emergency land in place | `MAV_CMD_NAV_LAND` at current position | Battery < 10% OR comms loss > 5 s with no RTL response |

---

## 5. Failsafe & Watchdog Contract

**Drone failsafes must be configured on the flight controller directly. Rover software is a secondary layer.**

### 5.1 Flight Controller Failsafes (Cube Orange+ / Pixhawk — configured in Mission Planner / QGC)

| Trigger | FC Action | Notes |
|---|---|---|
| RC signal loss > 1 s | RTL | RC is tertiary comms — loss means operator is relying on autonomous mode |
| Telemetry link (GCS) loss > 3 s | RTL | GCS = rover Jetson acting as GCS |
| Battery voltage < 3.5 V/cell (14.0 V for 4S) | RTL | Soft threshold |
| Battery voltage < 3.3 V/cell (13.2 V for 4S) | Land immediately | Hard threshold |
| Geofence breach | RTL | Geofence set to competition boundary + 20 m buffer |
| Crash detected (IMU spike) | Disarm immediately | |

### 5.2 Rover Software Failsafes (MAVSDK-ROS node)

| Trigger | Condition | Rover action |
|---|---|---|
| MAVLink heartbeat loss | No heartbeat for > 2 s | Send RTL via SiK 915 MHz fallback. Log event. Alert operator. |
| Drone battery status < 20% | `BATTERY_STATUS` MAVLink message | Issue RTL command. Notify operator. |
| Drone GPS accuracy > 5 m | `GPS_RAW_INT` hdop > 2.0 | Stop issuing new waypoints. Hold current mission. Alert operator. |
| ArUco detection conflict | Drone and ground detect different marker IDs | Ground detection takes precedence. Log both. Alert operator. |
| Wi-Fi lost during flight | ROS DDS heartbeat timeout | Switch to SiK telemetry-only mode. Video and map suspended. |
| Drone map ingestor crash | Node dies | Nav2 costmap layer cleared. Navigation continues on LiDAR only. |

### 5.3 Drone Docking Failsafes

| Scenario | Action |
|---|---|
| Drone misses pad by > 150 mm | Go-around — climb 3 m, reposition, retry. Max 3 attempts. |
| Pogo pins not making contact after landing | Operator must manually verify and reposition. Never auto-retry charge. |
| Drone not detected as docked but charge enabled | INA3221 current monitoring detects no charge current — disable GPIO 20, alert operator. |

---

## 6. Coordinate Frame Agreement

All drone data published in ROS 2 must be in the **rover world frame (ENU, origin = rover start position)**. The drone companion computer is responsible for this transformation using:
- Drone GPS (converted to UTM delta from rover start)
- Drone AHRS attitude

The rover Jetson does **not** do drone pose estimation — it trusts what the drone publishes.

| Topic | Frame | Transformation owner |
|---|---|---|
| `/drone/odometry` | `world` (ENU) | Drone RPi Zero 2W |
| `/drone/aruco_pose` | `world` (ENU) | Drone RPi Zero 2W |
| `/drone/map_tiles` | Geo-referenced (lat/lon) | `drone_map_ingestor` node on Jetson reprojects to `world` |

---

## 7. RF Coexistence

The drone and rover share the 5 GHz and 915 MHz spectrum. Both teams must coordinate on channel selection before any joint test.

| System | Frequency | Channel | Notes |
|---|---|---|---|
| Rover Wi-Fi AP | 5 GHz | Channel 36 (5180 MHz) | Fixed — do not change |
| Drone video stream | 5 GHz | Channel 36 | Same AP — connected as client |
| Drone SiK telemetry | 915 MHz | 25 (configurable) | Set same channel on both ends |
| Rover SiK radio | 915 MHz | 25 | Must match drone |
| RC transmitter (FrSky R9M) | 868 MHz | — | Non-overlapping with 915 MHz SiK |

**915 MHz and 868 MHz coexistence:** FrSky R9M operates at 868 MHz with frequency hopping. SiK operates at 915 MHz. 47 MHz separation — acceptable. Do not replace FrSky with a 915 MHz system without re-evaluating interference.
