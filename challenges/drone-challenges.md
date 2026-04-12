# Drone ↔ Rover: Challenges & Deep Dives
**Last updated:** 2026-04-12

---

## Challenge 1: MAVLink ↔ ROS 2 Bridge Latency and Reliability

### Problem
The drone publishes position, attitude, and ArUco detections that the rover Nav2 stack consumes. If bridge latency is high or the bridge crashes, the rover may:
- Navigate to a stale drone-detected ArUco pose (wrong goal)
- Lose drone position data and make decisions based on last-known state
- Have the MAVSDK-ROS node die silently while ROS 2 thinks it's still running

### Architecture Recommendation
Use **MAVROS** over raw MAVSDK-ROS for the rover side. MAVROS is a mature ROS 2 bridge with:
- Heartbeat monitoring (publishes `/mavros/state` with `connected` bool)
- Automatic reconnection on serial/UDP loss
- Latency reporting on each message type

**Drone side:** Use **MAVSDK** Python library on RPi Zero 2W. Send MAVLink over UDP to rover Jetson. Simpler than running full MAVROS on a Zero 2W.

### Latency Budget
| Data type | Max acceptable latency | Why |
|---|---|---|
| Drone position (for rover awareness) | 500 ms | Nav2 collision avoidance uses this — stale data not critical |
| ArUco pose (for rover goal) | 200 ms | Rover acts on this — stale data could send arm to wrong location |
| FPV video (for operator) | 300 ms | Human control loop tolerance |
| Scout map tiles | 5 s per tile | Nav2 costmap update rate — not real-time |
| Drone battery status | 5 s | Operator warning — not time-critical |

### Crash Recovery
MAVROS node must be launched with `respawn="true"` in the launch file. If MAVROS crashes:
1. `ros2` launch manager restarts it automatically.
2. During restart (~3 s), `/mavros/state.connected` = false.
3. A watchdog node monitors this topic. If false for > 5 s: log "drone_bridge_down", switch rover to ground-sensors-only mode.
4. Drone continues flying autonomously (FC failsafe handles comms loss — RTL).

---

## Challenge 2: Coordinate Frame Alignment Between Drone and Rover

### Problem
The drone has its own GPS and its own coordinate frame. The rover has its own EKF and its own world frame. When the drone reports an ArUco marker at position (X, Y, Z), that position is in the drone's coordinate frame — **not** the rover's world frame.

If the two frames are misaligned (different GPS origins, different heading conventions), the rover's arm will reach for the wrong location.

### Root Cause
- Rover world frame origin: first GPS fix at rover power-on, ENU convention.
- Drone world frame: depends on when FC was armed, heading from FC magnetometer.
- The two GPS receivers may give slightly different lat/lon for the same physical point.

### Solution: Shared Reference Point
1. **At mission start:** Rover operator uses a known benchmark point (competition provided GPS marker). Rover and drone are both commanded to report their GPS reading of this benchmark.
2. The difference between the two readings is the **GPS datum offset**. Store this offset.
3. Drone companion computer applies this offset when converting drone GPS to world frame before publishing to ROS 2.
4. This brings both frames into alignment to within GPS accuracy (~1–2 m, acceptable).

### Implementation in Drone RPi Zero 2W
```python
# At startup, receive datum offset from rover via ROS 2 parameter
datum_offset_x = rospy.get_param('/drone/datum_offset_x')
datum_offset_y = rospy.get_param('/drone/datum_offset_y')

# Apply when publishing ArUco pose
world_pose.position.x = drone_utm_x - rover_utm_origin_x + datum_offset_x
world_pose.position.y = drone_utm_y - rover_utm_origin_y + datum_offset_y
```

---

## Challenge 3: RF Interference Between Drone Motors and GPS/Comms

### Problem
Drone brushless motors with ESCs generate broadband RF noise from 1 MHz to 2 GHz. This can:
- Degrade GPS signal quality on both rover and drone (L1 = 1575.42 MHz)
- Interfere with 915 MHz SiK telemetry
- Corrupt 5 GHz Wi-Fi if noise floor is raised

### Measured Effect
GPS HDOP typically degrades from 1.2 to 2.8 when a hexacopter is at full throttle nearby. This is within our Mahalanobis gating threshold (3σ) but barely.

### Mitigation
1. **Ferrite cores on all ESC-to-motor cables.** Each motor cable gets a snap-on ferrite (Fair-Rite 0431173551) at the ESC exit.
2. **GPS antenna placement:** Rover GPS mast must be > 500 mm from drone landing pad. When drone is docked and motors are spinning (during pre-flight), this separation reduces coupling.
3. **SiK radio antenna placement:** Mount on rover rear, pointing away from drone pad. Directional antennas if budget allows.
4. **Power filtering on FC:** Add LC filter on FC power input (10 µH + 220 µF). ESC voltage spikes travel through shared power rails into FC and corrupt sensor readings.

---

## Challenge 4: Autonomous Landing on Moving / Vibrating Rover

### Problem
The drone must land on the rover's 300×300 mm pad. If the rover is parked with its engine running (motors idling), the chassis vibrates. Wind can affect approach. Landing requires precision of ±100 mm to hit the pad and ±50 mm to align pogo pins.

### Strategy: Vision-Assisted Landing
1. Attach an **AprilTag (6×6 cm, Tag36h11 family, ID 0)** to the centre of the drone landing pad.
2. Drone downward-facing camera detects this tag during landing approach.
3. Drone companion computer publishes tag pose via MAVSDK `SET_POSITION_TARGET_LOCAL_NED` for precision correction in final 3 m of descent.
4. Final approach: GPS → 3 m AGL → vision-based precision landing → touchdown.

### Rover Obligation
Rover must be **stationary** during drone landing. `drone-rover.md` already states this. The rover software must:
1. Stop all Nav2 goals before issuing a drone land command.
2. Set `cmd_vel` = 0 and hold.
3. Not issue any new Nav2 goals until drone landing is confirmed (FC reports `LANDED` state via MAVLink).

### Wind Edge Case
At competition, wind > 5 m/s makes precision landing unreliable. If wind causes repeated missed landings (> 3 attempts), the drone switches to manual landing mode — operator takes RC control for the final approach.

---

## Challenge 5: Drone ArUco Detection vs Rover Ground Detection Conflict

### Problem
Both the drone camera (high altitude, oblique angle) and the rover cameras (ground level, close range) may detect ArUco markers simultaneously. They may detect different markers (drone sees multiple from above) or the same marker at different computed positions (different camera angles = different pose estimates).

### Resolution Protocol (implement in `drone_aruco_relay` node)
1. **Ground detection takes precedence when rover is within 6.5 m of marker.** Ground pose estimate is more accurate at close range.
2. **Drone detection is used only to guide rover toward the marker** before the rover's cameras can see it.
3. If drone and ground disagree on marker ID: log both, alert operator, do not send any goal automatically. Require operator confirmation.
4. If drone and ground agree on ID but position differs by > 3 m: use ground estimate, log discrepancy.
5. Once rover ground detection locks (consecutive_frames ≥ 3), drone ArUco relay is silenced for that marker.

---

## Challenge 6: Drone Payload Release Mechanism

### Problem
The drone must drop a payload (marker, sample container, or flag) at a precise location. The release must be:
- Reliable: no mid-flight accidental release
- Precise: release within 0.5 m of target at 5 m altitude
- Recoverable: if release fails, drone must return with payload intact

### Mechanism Options
**Option A: Servo-actuated hook**
Single servo, spring-loaded hook releases payload. Commanded via MAVLink `DO_SET_SERVO`. Simple, proven in drone delivery.
Risk: Servo jitter from vibration may accidentally release. Add physical safety pin (remove before flight).

**Option B: Electromagnet**
Payload attaches via small neodymium magnet. Cut power to electromagnet to release. Zero moving parts.
Risk: Electromagnet requires constant power during flight (~2W). If power fails, payload drops immediately.

**Recommendation: Option A (servo hook + safety pin).**
- Safety pin prevents accidental release during takeoff/landing
- Operator removes pin after drone reaches 5 m AGL (operator confirmation step in mission)
- Servo is powered by BEC from FC — independent of companion computer

### Drop Accuracy
At 5 m altitude, a 0.5 m/s wind causes ~0.25 m drift during drop (assuming ~0.5 s fall time). At 10 m/s wind, drift = 5 m — unacceptable. 

**Mitigation:** Release at minimum practical altitude (3 m for 0.5 kg payload) and compensate for wind in the drop position calculation (measure wind speed from FC IMU velocity during hover, compute required offset).
