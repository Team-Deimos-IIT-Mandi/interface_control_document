# Mechanical ↔ Electrical Interface
**Boundary:** Physical construction — where structure meets silicon.
**Last updated:** 2026-04-12

---

## 1. Swerve Module Cable Routing

This is the hardest mechanical-electrical interface on the rover. Each steering joint rotates continuously. Cables must survive thousands of full rotations across a competition season without fatigue failure.

### 1.1 The Agreement

| Parameter | Mechanical Guarantees | Electrical Guarantees |
|---|---|---|
| Hollow steering shaft bore | ≥ 14 mm clear inner diameter through steering axis | Cable bundle ≤ 11 mm OD (3 mm clearance) |
| Steering travel | Hard stops at ±195°. Software limit at ±180°. | Cable system rated for ±200° continuous rotation, unlimited cycles |
| Cable method | Provides radiused bushing at shaft entry and exit (min 10 mm bend radius) | **Spiral-wound coiled cable** as primary. Slip ring as fallback (see below). |
| Coil cable spec | N/A | 22 AWG CAN-FD twisted pair + 18 AWG power (4 conductors max), shielded, coiled natural OD ≤ 30 mm |
| Slip ring fallback | Reserves 18 mm axial space above steering bearing for slip ring body | Slip ring: Adafruit 736 or Moog EC3848 — 4-circuit minimum, rated ≥ 2A per circuit |
| Conductor count per module | N/A — Electrical defines | CAN-H, CAN-L, PWR+, GND = 4 conductors. No more. Routing more through joint is banned. |

### 1.2 Cable Routing Path (per module)

```
Jetson CAN-FD bus → chassis cable tray → entry bushing (top of steering column)
→ coiled cable through hollow shaft → moteus r4.11 (steering) + AK80-9 (drive)
```

Power and CAN share the same coil. Signal integrity maintained by CAN-FD differential pair + shield drain tied to chassis GND at one end only.

---

## 2. Chassis Mounting Standards

All sub-systems bolt to a defined grid. Mechanical must not move mount locations without notifying Electrical.

| Item | Bolt pattern | Standoff height | Location on chassis |
|---|---|---|---|
| Jetson Orin NX carrier board | M3, 80 × 45 mm | 10 mm | Centre-forward, top plate |
| RPi 5 (arm co-proc) | M2.5, 58 × 49 mm | 8 mm | Centre-aft, top plate |
| RPi CM4 (science) | M2.5, 58 × 26 mm | 8 mm | Science bay, interior |
| PDU board | M4, 100 × 80 mm | 12 mm | Centre, under top plate |
| Drive battery cradle (×2) | M5, 200 × 80 mm | 0 mm (flush) | Low-centre, port + starboard |
| Compute battery | M4, 80 × 60 mm | 5 mm | Aft, low |
| Science payload bay | M4 corner mounts, 200 × 150 mm footprint | — | Forward-starboard |
| RPLIDAR A2M8 | M3, 4× holes, 30 mm circle | 15 mm (must clear 360° rotation) | Top plate, centre-forward — nothing within 50 mm radius at rotor height |
| Drone landing pad | Flat surface, 300 × 300 mm, M4 mounts | 0 mm | Aft top plate |
| Main E-STOP button | Panel cutout 22 mm diameter | — | Rear face of chassis, reachable without tools |

---

## 3. Keep-Out Zones

Mechanical must guarantee these volumes are clear. Electrical must not route cables through them.

| Zone | Volume | Reason |
|---|---|---|
| Arm swept volume (stow → full deploy) | 650 mm radius from arm base, 0–600 mm height | Arm crash protection |
| Drone rotor disk clearance | 200 mm above and 100 mm laterally from pad edge | Propwash + crash margin |
| LiDAR rotation plane | 50 mm above and below RPLIDAR rotor plane, 360° | Scan occlusion prevention |
| CAN-FD cable paths | 30 mm separation from any motor power cable run | EMI isolation |
| Battery terminals | 50 mm clear zone around each terminal | Short-circuit access prevention |

---

## 4. Thermal Management

| Heat source | Max dissipation | Mechanical's obligation | Electrical's obligation |
|---|---|---|---|
| Jetson Orin NX | 25 W peak | 80 × 80 mm heatsink footprint + 40 mm airflow clearance above | Do not seal enclosure without fan. Active fan if ambient > 35°C. |
| AK80-9 motor body | 15 W each continuous | Motor shell exposed to ambient — no foam or shroud against motor body | Monitor motor temperature via CAN telemetry. Throttle if > 80°C. |
| moteus r4.11 (×4) | 3 W each | Mount moteus boards to aluminium swerve bracket (acts as heatsink) | — |
| Li-ion drive packs | 5 W continuous | Packs in open aluminium cradle — no fully enclosed battery box | BMS monitors per-cell temperature. Alert if > 45°C. |
| Science spectrometer | 8 W | 2× 25 mm vent holes in science bay floor and lid | — |

---

## 5. Connector Standards

One connector family across the entire rover. No exceptions without ICD PR.

| Application | Connector type | Rationale |
|---|---|---|
| High-power (battery, drive motors) | XT60 | Industry standard, rated 60A, robust locking |
| Medium-power (arm, PDU outputs) | XT30 | Smaller, rated 30A |
| CAN-FD bus between modules | JST-GH 4-pin | Locking, compact, standard on moteus/AK80-9 |
| Signal connectors (sensors) | JST-SH 1.0 mm | Same as Pixhawk standard — interoperable with drone |
| E-STOP loop | Anderson SB50 | High-reliability, rated for repeated insertion cycles |

All connectors must be labelled with a heat-shrink sleeve marked with the destination (e.g., `FL_DRIVE`, `CM4_PWR`).

---

## 6. Wiring Harness Rules

1. Power cables and signal cables must run in separate cable trays or separated by ≥ 30 mm.
2. CAN-FD twisted pairs must not be untwisted more than 10 mm at termination points.
3. All power cables must be fused at source, not at destination.
4. Every cable must have a label at both ends.
5. No cable tie-wraps around rotating joints — use clip-on conduit only.
6. Minimum bend radius: 10× cable OD for power, 5× for signal.
