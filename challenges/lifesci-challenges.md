# Life Sciences: Challenges & Deep Dives
**Last updated:** 2026-04-12

---

## Challenge 1: Drill Interaction with Unknown Substrates

### Problem
The competition field has unknown soil composition. The drill may encounter:
- Loose sand (easy — low resistance, drill spins freely)
- Compacted clay (moderate — high resistance, risk of stall)
- Rock (stop immediately — damage to drill shaft and motor)

A naive PWM-on/off drill control will stall in clay and shatter the shaft on rock.

### Detection Strategy
Use CM4 GPIO current sensing on the drill motor circuit (INA219 on I2C 0x45):
- **Loose soil:** Current 0.5–1.5 A at 60% PWM
- **Compacted clay:** Current 2–4 A, possibly oscillating as motor labours
- **Rock:** Current spikes to > 5 A within < 100 ms

### Control Logic (implement in `science_mission.py`)
```
Phase 1 — Light probe (30% PWM, 2 s):
  If current < 0.5 A: "no ground contact" — lower arm further, retry
  If current 0.5–2 A: "soil detected" — proceed to Phase 2
  If current > 4 A: "rock/hard substrate" — abort, log, move to next site

Phase 2 — Full drill (60% PWM, 8 s):
  Monitor current every 100 ms
  If current > 5 A for > 200 ms: stop, retract, retry once at 20% offset position
  If current drops to < 0.3 A mid-drill: "drill broke free" — complete cycle normally
  If stall (current > 4 A for > 1 s): stop, log "hard_layer", continue with surface sample only

Phase 3 — Retract:
  Run drill at 30% PWM in reverse for 2 s while retracting arm (clears cuttings from bore)
```

---

## Challenge 2: Spectrometer Calibration Under Field Conditions

### Problem
Optical spectrometers require calibration against a known reference. In a lab, this is done with a calibration tile. At competition, the reference may not be available, and changing sunlight conditions alter readings.

### Mitigation
1. **Dark calibration:** Take a dark scan (no light) immediately before each science site measurement. Subtract dark from sample reading. CM4 handles this automatically at the start of each science sequence.
2. **Reference tile:** Carry a small spectralon (PTFE-based) reference tile in the science bay. Science mission takes a reference scan before lowering probe, then takes soil scan. Report ratio (relative reflectance) rather than absolute.
3. **Sunlight rejection:** Schedule spectrometer scans immediately after drill completes (soil sample is below surface level — less direct sunlight interference). If ambient light is very high (midday), the scan SNR degrades — log `ambient_light_level` from the spectrometer's built-in light sensor if available.
4. **Fallback if calibration impossible:** Log raw counts + timestamp + GPS. Post-process after competition.

---

## Challenge 3: Microscope Focus at Variable Ground Distance

### Problem
The arm lowers the probe to ground contact, but ground height varies by ±20 mm across a site. The microscope has a fixed focal length. If it's 5 mm too high or low, images are blurry.

### Mitigation
1. **ToF pre-measurement:** Before lowering arm, use MTF-01 ToF sensor (already on the rover) to measure ground height at the science site. Adjust arm "science_probe" target height accordingly.
2. **Multi-frame bracketing:** Capture 5 frames with arm at slightly different heights (±5 mm steps from nominal). Save all 5. Select sharpest frame in post-processing using Laplacian variance metric (implementable in CM4 OpenCV).
3. **Mechanical focus lock:** Design microscope mount with a thumbscrew focus adjustment. Set focus for the nominal probe-to-ground distance at assembly. This is a one-time calibration, not adjusted in field.

---

## Challenge 4: I2C Address Conflicts in Science Bay

### Problem
Moisture (0x28), pH (0x36), and temperature (0x40) sensors are on the same CM4 I2C bus. If any two sensors share an address (possible with off-the-shelf modules), they will collide and produce garbage data.

### Verification
Before integration: connect each sensor individually and confirm its I2C address with `i2cdetect -y 1`. Document actual addresses — datasheet addresses and actual addresses sometimes differ due to address pin strapping.

### Mitigation if Collision Occurs
Option A: Use I2C address-programmable variants of sensors (most modules have solder jumpers for address selection).
Option B: Add TCA9548A I2C multiplexer — 8 independent channels, each sensor on its own channel. CM4 selects the active channel via I2C command to the mux (address 0x70).

**Recommendation:** Use the TCA9548A regardless of collisions — it adds robustness. If one sensor's I2C line gets stuck LOW (common fault), the mux isolates it without pulling the entire bus low.

---

## Challenge 5: Science Data Integrity Under System Reset

### Problem
CM4 may reboot mid-mission (watchdog reset, power glitch). If `science_results.csv` is being written at the moment of reset, the file may be corrupt (incomplete row, missing newline, buffer not flushed).

### Mitigation
1. **Atomic writes:** Each science row is written to a temporary file first (`science_results.csv.tmp`), then renamed to the final file on completion. Rename is atomic on Linux (ext4 filesystem). An interrupted write leaves the .tmp file, not a corrupt .csv.
2. **Flush after every write:** Call `file.flush()` + `os.fsync(file.fileno())` after each row. Ensures data reaches disk before acknowledgement.
3. **Write to two locations:** Write `science_results.csv` to both CM4 SD card and Jetson (over the UART bridge). If CM4 SD card fails, Jetson has the data.
4. **Incremental backup:** Every 60 seconds, CM4 copies `science_results.csv` to `/tmp/science_backup.csv`. If SD card is corrupt, recover from backup.

---

## Challenge 6: Drill Shaft Replacement in Field

### Problem
Drill shafts break. At competition, you may need to swap one in < 5 minutes without tools you forgot to bring.

### Design Requirements for Mechanical
1. Drill shaft must be hex-drive (6.35 mm hex shank) — any cordless drill bit fits.
2. Chuck must be a quick-release collet (quarter-turn, no tools needed).
3. Spare drill bits (3 minimum) carried in a labelled slot inside science bay lid.
4. Motor coupling must be accessible without removing the science bay from the rover.

### Procedure Card
Laminate a procedure card and attach it to the inside of the science bay lid:
```
DRILL BIT REPLACEMENT (< 3 minutes):
1. Ensure E-STOP is pressed (red button, rear of rover)
2. Open science bay lid (4× thumb screws)
3. Press collet release button, pull old bit out
4. Insert new bit, push until click
5. Close bay lid, release E-STOP
6. Test drill: ros2 service call /science/drill_test
```
