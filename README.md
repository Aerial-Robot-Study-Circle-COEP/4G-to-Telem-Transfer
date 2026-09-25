# NEMA 17 — 60° Stepper Indexing with Raspberry Pi 4B + A4988

Drive a NEMA 17 stepper motor in precise 60° increments using a Raspberry Pi 4B and an A4988 driver breakout board (the "STEPPER USE 9V/1A POWER" style module with E/D/S signal pins).

## Purpose

This mechanism is used as a **payload release actuator on a delivery drone**, indexing a six-position payload carousel/drum in 60° steps to sequentially drop up to **five 200 g parcels**, one per index. Each command (`stepper60.py N`) advances the mechanism by `N` positions over telemetry from the companion computer, releasing one parcel bay at a time. Because releases are discrete positions rather than continuous motion, open-loop stepping is sufficient, and the absolute-step-tracking in the control code ensures the mechanism returns to exact zero after a full 360° cycle, so alignment doesn't drift across repeated delivery runs.

## Purpose: Drone Payload-Drop Actuator

This mechanism is designed as the release actuator for a multi-payload delivery drone. A single NEMA 17, geared to a carousel/hub holding **5 parcels of 200 g each (1 kg total payload)**, indexes 60° per release command — one full 360° rotation cycles through all 5 drop positions plus a return-to-home/locked position, releasing exactly one parcel per index.

Key implications for this use case:

- **Ground-triggered, not continuous motion.** The motor sits idle (or holds position) between drops and only moves on command from the flight controller / companion computer over telemetry, matching the single-index-per-call design of `stepper60.py`.
- **Positional accuracy matters more than speed.** A missed or partial index could jam the mechanism or drop two parcels at once mid-air, so the current limit (Vref) and microstepping should be tuned for reliable full-torque indexing under the load of an off-balance, partially-loaded carousel — not for top speed.
- **Weight and power budget.** A 1 kg total payload plus the actuator, driver, wiring, and 9 V supply all add to the drone's takeoff weight — factor this into the airframe's payload capacity separately from the electronics design below.
- **Fail-safe consideration.** Since the code releases the motor (disables the driver) after each move by default, an unloaded/de-energized carousel could potentially drift under vibration or gusts. If holding torque between drops is required to prevent unintended releases mid-flight, keep the driver enabled (see the "Holding torque" note under Troubleshooting/Notes) at the cost of extra heat and current draw — test this tradeoff on the bench before flight.
- **Command source.** "Via telemetry" in this project means index commands arrive from the drone's companion computer/GCS link rather than a human at a terminal — wire `stepper60.py` into that command channel (e.g. triggered by a MAVLink command, an MQTT message, or a serial packet) rather than running it interactively in flight.

## Hardware

- Raspberry Pi 4B
- NEMA 17 stepper motor
- A4988 breakout board (3-pin DIP microstep selector, E/D/S signal header, JST motor connector)
- 9 V / 1 A DC power supply (per the board's silkscreen — do not use 12 V on this board)
- 100 µF electrolytic capacitor across the 9V/GND power input (if not already fitted)
- Multimeter (for setting the current limit)
- Small plastic/ceramic screwdriver (for the Vref trim pot)
- Jumper wires

## Wiring

All grounds must be common. Power off while wiring.

| Board pin | Connect to | Pi GPIO (BCM) | Pi physical pin |
|---|---|---|---|
| **E** (ENABLE) | Pi GPIO | GPIO 16 | 36 |
| **D** (DIR) | Pi GPIO | GPIO 21 | 40 |
| **S** (STEP) | Pi GPIO | GPIO 20 | 38 |
| **5V** | Pi 5V | — | 2 |
| **GND** (logic) | Pi GND | — | 6 |
| **9V** | + of 9 V supply | — | — |
| **GND** (power) | − of 9 V supply, common with Pi GND | — | — |
| Motor JST | NEMA 17 motor cable | — | — |

> Verify pin labels against your own board's silkscreen before connecting — wire by the printed letter (E/D/S), not by position.

## Microstepping

Set all three DIP switches to **ON** for 1/16 microstepping (3200 steps/revolution), unless the table printed on your board specifies different switch positions for 1/16. This must match `STEPS_PER_REV` in the code.

## Setting the current limit (Vref)

Do this with the **motor unplugged**.

1. Power the board (9V, 5V, GND connected).
2. Multimeter on DC volts. Black probe on GND, red probe on the small trim-pot screw on top of the A4988 chip.
3. Compute the target: `Vref = I_limit × 8 × Rsense`
   - Read `Rsense` off the two small resistors next to the chip (commonly 0.1 Ω or 0.068 Ω).
   - Set `I_limit` to roughly 70% of the motor's rated current, and no more than ~0.7 A given the 1 A supply.

| Motor rated | Rsense 0.068 Ω | Rsense 0.1 Ω |
|---|---|---|
| 1.0 A | 0.38 V | 0.56 V |
| 1.5 A | 0.54 V | 0.80 V |
| 1.7 A | 0.65 V | 0.96 V |

4. Turn the screw slowly with a plastic screwdriver until the target voltage is reached. Power off before plugging in the motor.

**Never connect or disconnect the motor while powered.**

## Software setup

```bash
sudo apt update
sudo apt install python3-rpi.gpio
```

Save `stepper60.py` (included in this repo) and run:

```bash
python3 stepper60.py 1     # one 60° index forward
python3 stepper60.py 6     # full revolution (6 × 60°), returns exactly to start
python3 stepper60.py -2    # two indexes in reverse
```

### How it avoids drift

3200 steps/rev is not evenly divisible by 6 (60° steps), so rounding each individual move would accumulate error over repeated indexing. The script instead tracks the **absolute cumulative target** in steps and rounds that, so after 6 indexes the motor lands on exactly 3200 steps (360°) with zero net drift.

### Adjusting speed

Speed is controlled by `STEP_DELAY` in the script (seconds of HIGH + seconds of LOW per step).

- Steps/second = `1 / (2 × STEP_DELAY)`
- RPM = `steps_per_second × 60 / STEPS_PER_REV`

| STEP_DELAY | Approx. speed |
|---|---|
| 0.004 | ~2.3 RPM |
| 0.002 | ~4.7 RPM |
| 0.001 (default) | ~9.4 RPM |
| 0.0005 | ~19 RPM |

Lower `STEP_DELAY` = faster, but pure `time.sleep` timing on a Pi gets unreliable much above ~40 RPM at 1/16 microstepping — use `pigpio` waveforms or hardware PWM for smoother high-speed motion.

## Switching to telemetry-link SSH

If you want to command this actuator over the telemetry radio link instead of Wi-Fi/4G (for field/flight use), see [4G-to-Telem-Transfer](https://github.com/Aerial-Robot-Study-Circle-COEP/4G-to-Telem-Transfer) — it sets up PPP over a SiK-style telemetry radio pair so you can `ssh` to the companion Pi the same way, just over the radio link instead.

## First test

1. Confirm wiring, DIP switches, and Vref are all set. Motor plugged in, power off.
2. Mark the shaft (tape or similar) as a visual reference.
3. Power on, run `python3 stepper60.py 1` — shaft should turn 60°.
4. Run `python3 stepper60.py 6` — shaft should return exactly to the marked position.

## Troubleshooting

| Symptom | Likely fix |
|---|---|
| Nothing moves | Try flipping `EN_ACTIVE_LOW` in the script; check 9 V present and common ground |
| Motor buzzes but doesn't turn | Swap the two wires of one motor coil (power off first), or increase `STEP_DELAY` |
| Wrong angle per index | Confirm all DIP switches ON and `STEPS_PER_REV = 3200` matches |
| Motor skips steps | Increase Vref slightly (within motor rating) or increase `STEP_DELAY` |
| Driver runs very hot | Decrease Vref |

## Notes / caveats

- This board is rated for **9 V / 1 A** — do not substitute a 12 V supply.
- Exact pinout/DIP tables vary slightly between A4988 breakout clones — always confirm against your own board's silkscreen before powering up.
- No feedback/encoder is used; this is open-loop stepping. Missed steps under excess load or insufficient current will not be detected by the software.
