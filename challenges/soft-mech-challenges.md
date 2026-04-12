# Soft ↔ Mech: Challenges & Deep Dives
**Last updated:** 2026-04-12

---

## Challenge 1: Swerve Inverse Kinematics on Rough Terrain

### Why Swerve Is Hard in Software
Differential drive has one kinematic equation. Swerve has 8 (one per motor, two per module). The swerve inverse kinematics solver must run at 100 Hz and compute — for every target `cmd_vel` (vx, vy, ω) — the exact drive speed and steer angle for each of the 4 modules.

The standard swerve IK formula for module `i` at offset `(xi, yi)` from CoM:

```
vx_i = vx - ω * yi
vy_i = vy + ω * xi

wheel_speed_i = sqrt(vx_i² + vy_i²)
steer_angle_i = atan2(vy_i, vx_i)
```

This is clean on flat ground. On rough terrain, three complications arise:

### Complication 1: Wheel Slip
If a wheel is slipping (optical flow vs wheel encoder disagrees), the IK's assumed wheel speed is wrong. The rover yaws unexpectedly.

**Mitigation:** The slip detection node (already designed in the sensor fusion architecture) publishes a per-wheel slip flag. When slip is detected on a module, reduce that module's commanded speed by the slip ratio estimate. This is a soft correction — it degrades trajectory accuracy but prevents spin-out.

### Complication 2: Ground Contact Loss
On a rock, one or more wheels may lose ground contact. A wheel in the air accelerates freely — its encoder reports high speed but it's contributing zero traction.

**Mitigation:** Monitor each AK80-9's current feedback. A wheel in the air draws significantly less current than one under load. If current < 20% of expected load current for > 200 ms at non-zero command: flag as "airborne", reduce commanded speed to 0 on that module, redistribute traction to other modules.

### Complication 3: Steering Angle Optimisation
When reversing direction, the naïve IK tells the steering motor to rotate 180°. A 180° steer takes ~500 ms at maximum steer rate — the rover is stationary for half a second every time it reverses.

**Mitigation:** Implement "steer optimisation" — if the computed steer angle differs from the current angle by > 90°, flip the drive direction and command a steer angle of `target ± 180°` instead. This reduces maximum steer travel to 90° at the cost of reversing the drive wheel. Already a standard technique in FRC swerve libraries.

---

## Challenge 2: URDF Accuracy Under Dynamic Loads

### Problem
The URDF specifies fixed link lengths and joint positions. In reality, the 50 kg rover on rough terrain deflects. Arm links deflect under payload. Swerve module suspension (if any) changes wheel height relative to chassis.

If the URDF is wrong by even 5 mm at the end-effector, MoveIt2's collision avoidance may allow a trajectory that physically crashes the arm into the chassis or terrain.

### Mitigation
1. **Model suspension deflection:** If passive suspension is used, add a small translational joint in the URDF for each module's suspension travel. Set joint limits to match physical spring travel. Set this joint as non-actuated (read-only from a force estimate).
2. **Arm link stiffness:** Use sufficiently rigid arm links (minimum 6 mm wall thickness CF tubes for links > 200 mm). Deflection at end-effector under 2 kg payload should be < 2 mm. Verify experimentally.
3. **MoveIt2 collision margin:** Set `robot_description_planning.default_robot_padding` to 15 mm in the MoveIt2 config. This adds a 15 mm safety buffer around all collision bodies, absorbing deflection and TF uncertainty.

---

## Challenge 3: Arm Stow Constraint During Navigation

### Problem
Nav2 plans paths based on the rover's footprint polygon. When the arm is stowed, the footprint is the rover chassis. When the arm is deployed or partially deployed, the footprint expands — potentially into the path Nav2 just planned.

Additionally, if the rover drives with the arm not fully stowed (e.g., partially retracted after a manipulation task), the arm may collide with terrain features the rover drives over.

### Mitigation
1. **Stow enforcement:** The `pick_place_server.py` node must confirm arm is in named pose `stow` before publishing any Nav2 goal. MoveIt2 must confirm `stow` is reached before the server responds success.
2. **Dynamic footprint update:** `slope_detector.py` already monitors IMU for terrain safety. Add arm state monitoring: if arm is not in `stow`, publish an expanded footprint to Nav2 covering the arm's current swept volume (computed from forward kinematics at current joint angles).
3. **Hard constraint:** ros2_control must reject any `cmd_vel` command if the arm is not in `stow` pose and `cmd_vel` magnitude > 0.1 m/s. This is a software interlock, not a hardware one. Implement in the swerve controller node.

---

## Challenge 4: Measuring Rover CoM After Build

### Problem
The URDF's mass and inertia values are used by:
- MoveIt2 for dynamic planning (Coriolis and centrifugal effects at high speed)
- Nav2 MPPI controller for predicting rover trajectory
- Tip-over detection (IMU roll/pitch thresholds)

Wrong CoM = wrong controller tuning = unexpected behaviour at speed.

### Measurement Protocol
1. **Weighing:** Place rover on 4 separate scales (one per module). Sum = total mass. CoM X = (sum of front scale readings × distance to front) / total mass. Repeat for Y.
2. **CoM height:** Tilt rover to a known angle on a ramp. Measure the angle at which the rover would tip (with safety rope). Back-calculate CoM Z from tipping angle and wheelbase.
3. **Arm contribution:** Measure with arm in stow and in deploy. The difference tells you the arm's contribution to CoM shift. This is the input to the dynamic footprint calculation.
4. **Science payload:** Measure with science bay empty and full. Full = all 4 sample containers loaded.

**Deadline:** CoM measurements must be completed within 3 days of rover chassis completion. All values entered into `soft-mech.md` before any autonomous outdoor testing.

---

## Challenge 5: Swerve Module Zero-Point Calibration

### Problem
The AS5600 magnetic absolute encoder reports angles in 0–360°. The "steer zero" position (wheel pointing forward) must be determined at assembly time and stored. If the zero point drifts or is wrong, the rover will drive crooked even with a zero steering command.

### Calibration Procedure
1. At assembly: place rover on flat surface. Use a laser line to align all 4 wheels perfectly parallel.
2. Read AS5600 angle for each module via moteus `d pos 0 0 0` command (read raw encoder position).
3. Store these 4 angle offsets in `config/swerve_calibration.yaml`.
4. Software subtracts the offset before commanding each steering motor.
5. **Verification:** After calibration, command `cmd_vel` with pure vx=0.1 m/s, vy=0, ω=0. Rover should travel in a straight line. If it yaws, re-calibrate the worst module.

**Recalibrate after:** Any disassembly of a swerve module, any magnet replacement, any chassis crash that could have shifted the magnet.
