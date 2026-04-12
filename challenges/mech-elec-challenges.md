# Mech ↔ Elec: Challenges & Deep Dives
**Last updated:** 2026-04-12

---

## Challenge 1: Cable Routing Through Rotating Steering Joints

### Why This Is Hard
A swerve module's steering axis rotates ±180°. Any cable passing through it accumulates fatigue cycles. At 1 revolution per 5 seconds (aggressive manoeuvring), a 4-hour competition day accumulates ~3000 rotation cycles on each module. Standard wire will crack at the insulation within 500 cycles if not designed for flex.

### Option A: Spiral-Wound Coiled Cable
**How it works:** The cable is coiled like a telephone cord and enters the hollow steering shaft. As the shaft rotates, the coil winds tighter or looser, absorbing ±180° of rotation without any bending at a fixed point.

**Pros:** Simple, no rotating contacts, no signal loss, cheap.
**Cons:** Coil can tangle if it overwinds (rotation > 360°). Must implement software steering limit at ±180° strictly. Coil OD must fit within bore.

**Recommended spec:** 4-conductor shielded coil cable, 22 AWG signal + 18 AWG power, natural coil OD ≤ 25 mm, extended length ≥ 300 mm, rated for > 100,000 flex cycles. Source: IGUS Chainflex CF9 series or equivalent.

**Critical assembly note:** Coil must be free-hanging inside the shaft with no constraint points. If a cable tie-wrap or zip tie pinches the coil at any point, it will fail at that pinch within hours.

### Option B: Slip Ring
**How it works:** Fixed brushes contact rotating rings. Signal passes through regardless of rotation angle. Supports unlimited rotation.

**Pros:** No rotation limit — could simplify steering controller (no unwinding logic needed).
**Cons:** Brushes wear over time. Dust ingress at competition kills contact quality. Contact resistance adds noise to CAN-FD differential pair. 4-circuit slip rings with adequate current rating are expensive (Moog EC3848 ≈ $300+ per module).

**Recommended only if:** Mechanical cannot guarantee ±180° hard stops with < 5° tolerance, OR the team wants to support continuous rotation swerve in future.

**If slip ring is chosen:** Shield the slip ring housing against dust ingress (silicone seal). Use gold contacts, not copper. Verify CAN-FD signal integrity across slip ring with oscilloscope before field deployment.

### Decision Required
Team must decide between Option A and Option B before manufacturing steering column. This cannot be changed after the hollow shaft is machined.

**Recommendation: Option A (coiled cable)** for first build. Simpler, cheaper, more reliable in dust. Switch to slip ring only if ±180° proves operationally limiting.

---

## Challenge 2: Thermal Management Under Field Conditions

### Environment
ARC/URC competition fields are outdoors. Ambient temperature can reach 35–45°C. The rover chassis may absorb solar radiation, raising internal temperature by an additional 10–15°C above ambient. Compute boards and motor controllers are rated for typically 0–70°C junction temperature.

### Jetson Orin NX Thermal Risk
At 45°C ambient + solar gain, the Jetson enclosure internal temperature could reach 55–60°C before any load. Under full GPU + CPU load, the Orin NX junction can hit 80°C, triggering thermal throttling which reduces Nav2 and vision update rates — exactly when you need full performance.

**Mitigation:**
1. Mount Orin NX carrier board with heatsink exposed to airflow, not sealed in a box.
2. Add a 40×40×10 mm heat pipe between Orin NX module and heatsink.
3. If enclosure is needed for dust: use a sealed enclosure with a 92mm fan recirculating air over a finned heatsink with a rubber-gasketed vent.
4. Monitor `sudo tegrastats` junction temperature during field tests. If > 75°C at idle, the thermal design is failing.

### AK80-9 Motor Thermal Risk
AK80-9 motors dissipate heat through their housing. On a 45°C day, continuous driving at 40% load will raise motor case temperature to 55–65°C. The motor is rated for 80°C case temperature — there is margin, but it will be consumed fast on loose terrain where slip causes higher current draw.

**Mitigation:**
1. Do not cover motor housings with any insulating material.
2. Orient motors so airflow from rover motion passes over the housing (fins facing outward).
3. Monitor motor temperature via CAN telemetry (AK80-9 reports motor temperature). Log it.
4. Software throttles motor power if temperature > 75°C (see elec-soft failsafes).

### Li-ion Battery Thermal Risk
21700 Li-ion cells are safer than LiPo but still lose capacity and risk thermal runaway above 45°C cell temperature. BMS must monitor per-cell temperature.

**Mitigation:**
1. Do not place batteries directly on top plate (solar radiation).
2. Place batteries low in chassis — they are naturally shaded by chassis walls.
3. Never charge batteries in direct sunlight. Charge in shade or indoors.
4. If cell temperature > 45°C: do not charge. If > 55°C: BMS disconnects automatically.

---

## Challenge 3: EMI from Motor Controllers Affecting Sensors

### Problem
CAN-FD motor controllers switching at high frequency generate electromagnetic interference. The ICM-20948 IMU is SPI-connected at 7 MHz — a well-designed differential interface — but the magnetometer inside the ICM-20948 is susceptible to the magnetic fields generated by high-current motor wiring.

The magnetometer is not used in the primary EKF (gyro + accel integration dominates), but if magnetometer data corrupts, it can cause heading drift under certain EKF configurations.

### Mitigation
1. Route motor power cables (18 AWG, carrying up to 20A per module) away from IMU location. Minimum 30 mm separation from IMU package.
2. Add ferrite clamp to motor power cable bundles at chassis exit point.
3. Use twisted pair for all CAN-FD signal cables.
4. Mount IMU on vibration-isolated standoffs (silicone grommets, 10 Shore A) — reduces both EMI coupling and vibration noise.
5. If magnetometer data is needed: calibrate on-field with motors running (hard/soft iron compensation at operating current levels).

---

## Challenge 4: Connector Reliability in Dust and Vibration

### Problem
ARC competition fields have fine dust that infiltrates connectors. Vibration from rover traversal causes micro-fretting corrosion on connector contacts, increasing contact resistance over time. A connector that works on day 1 may fail silently on day 3.

### Mitigation
1. Use gold-plated contacts for all signal connectors (JST-SH 1.0 mm) — gold resists corrosion.
2. Apply Nyogel 760G or similar non-conductive connector lubricant to signal connector contacts before field deployment.
3. All XT60/XT30 power connectors must click fully seated — perform pull test (> 10 N) after each assembly.
4. After each competition day: visually inspect all connectors for dust intrusion. Clean with compressed air.
5. Carry spare connectors and pre-crimped pigtails for every connector type used on the rover.
