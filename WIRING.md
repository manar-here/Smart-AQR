# Wiring Guide

## Components

- 1x ESP32 dev board (e.g. ESP32-WROOM-32 DevKit)
- 4x hobby servos (e.g. SG90 / MG90S)
- 1x HC-SR04 ultrasonic distance sensor
- 1x external 5V power supply for the servos (a separate battery pack or bench supply — **not** the ESP32's own 5V/3.3V regulator, see below)
- Jumper wires, breadboard or perfboard
- 1x 1kΩ resistor + 1x 2kΩ resistor (for the HC-SR04 echo line, see below)

## Default Pin Map

These match the defaults in `esp32/quadruped_robot/config.h` — change either side to match your build.

| Signal | ESP32 GPIO |
|---|---|
| Servo 1 (e.g. Front-Left) | GPIO 13 |
| Servo 2 (e.g. Front-Right) | GPIO 12 ⚠️ see note below |
| Servo 3 (e.g. Back-Left) | GPIO 14 |
| Servo 4 (e.g. Back-Right) | GPIO 27 |
| HC-SR04 `TRIG` | GPIO 5 |
| HC-SR04 `ECHO` (through divider, see below) | GPIO 18 |

> **GPIO 12 note:** this is one of the ESP32's boot "strapping" pins (it selects flash voltage at reset). It's fine as a regular output in this project since nothing external holds it high before boot — but if you ever see random resets or boot/Wi-Fi issues, move Servo 2 to another free pin (e.g. `GPIO 4` or `GPIO 16`) and update `config.h` to match.

## ⚠️ Power Planning — Yes, You Need a Separate Supply

**A separate 5V supply for the servos is not optional.** When the ESP32 is powered from a computer's USB port or a generic phone charger, that source typically only provides ~500mA–1A. Four servos moving together can draw over 1A, with brief spikes much higher if a servo stalls. Worse, a moving servo puts current *pulses* on the 5V line — if the ESP32 shares that same line, those pulses cause a voltage sag ("brownout") that can silently reset or freeze the microcontroller mid-motion. This is the single most common failure mode in small robot builds, so give the servos their own supply and tie its ground to the ESP32's `GND`.

| Option | Voltage | Recommended rating | Notes |
|---|---|---|---|
| **USB power bank, dedicated to the servo rail** ✅ | 5V | 2A+ | Easiest and safest choice — already regulated, cheap, portable |
| 4x AA rechargeable NiMH pack | 4.8V | ~2000mAh capacity | Very common in beginner kits; voltage sags as the pack drains |
| 2S LiPo (7.4V) + a 5V UBEC/buck converter | 5V (after conversion) | UBEC rated 3A+ | Higher capacity, rechargeable, but needs safe LiPo handling/charging |
| Bench power supply | 5V | 2–3A | Best for calibration/testing, not portable |

The ESP32 itself can stay on its own USB cable (for programming/Serial Monitor), or share the same 5V rail as the servos as long as that supply's total rating covers everything with margin.

**Golden rule: every ground must be common** — ESP32 `GND`, the servo supply's `GND`, and the HC-SR04 `GND` all need to be tied together, or the PWM signal has no valid reference and servos will glitch or not move at all.

## ⚠️ HC-SR04 Echo Pin — Voltage Divider Required

The HC-SR04 runs on 5V and its `ECHO` pin outputs a **5V** signal. ESP32 GPIOs are **3.3V-only** and can be damaged by a direct 5V connection. Build a simple two-resistor divider between `ECHO` and the ESP32 pin:

```
HC-SR04 ECHO ──[1kΩ]──┬──[2kΩ]── GND
                       │
                    ESP32 GPIO 18
```

This brings the ~5V signal down to roughly 3.3V — specifically, `5V × (2kΩ ÷ (1kΩ + 2kΩ)) ≈ 3.33V`, safely within the ESP32's 3.3V logic range. `TRIG` can connect directly to the ESP32 (it's an input to the sensor, driven at 3.3V, which the HC-SR04 reads as logic-high fine).

## Connections

| HC-SR04 | ESP32 / Power |
|---|---|
| `VCC` | 5V supply (same rail as servos, common ground) |
| `TRIG` | GPIO 5 |
| `ECHO` | GPIO 18, via the voltage divider above |
| `GND` | Common ground |

| Servo | Power / Signal |
|---|---|
| Red (V+) | External 5V supply |
| Brown/Black (GND) | Common ground |
| Orange/Yellow (Signal) | Its GPIO from the pin map above |

## After Wiring

1. Connect all grounds together **first**, before anything else.
2. Build and connect the ECHO voltage divider before wiring the HC-SR04 itself.
3. If you have a multimeter, verify GPIO 18 reads ~3.3V (not ~5V) while the sensor is triggered, before connecting it to the ESP32.
4. Double-check every ground is common before powering on.
5. Flash `esp32/quadruped_robot/quadruped_robot.ino` (see the root [README](../README.md) for library requirements).
6. Open the Serial Monitor at 115200 baud to confirm Wi-Fi connects and note the IP address.
7. With the robot safely propped up off the ground (legs can't catch on anything), test each pose from the dashboard's Manual Control page before attempting a walking gait.
