# MiniPos Az-El Rotator — Ground Station Documentation

**Project:** MiniPos Ground Station Positioner 
**Team:** Open Orbit CubeSat Team, Colorado State University  
**Revision:** 1.0 — May 2026

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Bill of Materials](#bill-of-materials)
3. [Electrical Interfaces](#electrical-interfaces)
4. [Pin Assignments](#pin-assignments)
5. [Wiring Notes](#wiring-notes)
6. [Firmware Architecture](#firmware-architecture)
7. [Software Stack](#software-stack)
8. [Operating Procedure](#operating-procedure)
9. [Known Limitations & Future Work](#known-limitations--future-work)

---

## System Overview

MiniPos is a belt-driven azimuth-elevation antenna rotator designed for CubeSat ground station use. It uses two DC geared motors with hall encoders, controlled by an ESP32 over Bluetooth SPP, and interfaces with GPredict via the Hamlib ROT2Prog protocol.

**Drive System:**
- Motor internal gear ratio: 131:1
- Belt pulley ratio: 6.667:1 (20:3 — 1.2" drive pulley, 8" driven pulley)
- Total reduction: ~873:1
- Output speed (no-load): ~12 RPM (~72°/sec)

**Structural:**
- FDM printed structure, ~400g total print mass
- 50×65mm deep groove ball bearings at rotation axes
- Polyurethane belt drive, cut 5–10% shorter than loop length to maintain tension

---

## Bill of Materials

| Part # | Name               | Section       | Qty | Source   | Cost ($) | FDM Mass (g) |
|--------|--------------------|---------------|-----|----------|----------|--------------|
| 1      | 1.2" Pulley Wheel  | Pulley Drive  | 2   | FDM      | —        | 5            |
| 2      | 8" Pulley Wheel    | Pulley Drive  | 2   | FDM      | —        | 78           |
| 3      | Polyurethane Belt  | Pulley Drive  | 2   | Amazon   | 10       | —            |
| 4      | Motor Carrier      | Pulley Drive  | 2   | FDM      | —        | 50           |
| 5      | Motor Collar       | Pulley Drive  | 2   | FDM      | —        | 7            |
| 6      | Geared Motor       | Pulley Drive  | 2   | DigiKey  | 33       | —            |
| 7      | 50×65mm Bearing    | Bearing Assy. | 3   | Amazon   | 20       | —            |
| 8      | Bearing Hub        | Bearing Assy. | 3   | FDM      | —        | 8            |
| 9      | Bearing Collar     | Bearing Assy. | 1   | FDM      | —        | 17           |
| 10     | Bearing Retainer S | Bearing Assy. | 3   | FDM      | —        | 2            |
| 11     | Bearing Retainer L | Bearing Assy. | 3   | FDM      | —        | 2            |
| 12     | Upright Mounts     | Structure     | 1   | FDM      | —        | 35           |
| 13     | Shaft Coupler      | Structure     | 3   | FDM      | —        | 8            |
| 14     | 12V PSU            | Power         | 1   | Amazon   | 20       | —            |
| 15     | 12V→5V Step Down   | Power         | 1   | Amazon   | 12       | —            |
| 16     | L298 Motor Driver  | Control       | 1   | Amazon   | 25       | —            |
| 17     | ESP32-WROOM-32UE   | Control       | 1   | DigiKey  | 10       | —            |

**Approximate BOM Cost: ~$200**  
**Approximate FDM Print Mass: ~400g**

### Bearing Assembly Order (Right to Left)

`8" Pulley Wheel → Bearing Hub → Bearing Retainer L → Bearing → Bearing Retainer S → Bearing Collar`

Part #s: `2 — 8 — 11 — 7 — 10 — 9`

---

## Electrical Interfaces

### Power Supply (ALITOVE 12V 10A)

| Terminal | Connection |
|----------|------------|
| L (AC)   | Mains Live |
| N (AC)   | Mains Neutral |
| ⏚ (GND) | Chassis earth (enclosure if present) |
| -V       | System GND rail |
| +V       | 12V rail → L298 V+, Step-down input |
| ADJ      | Leave at default unless voltage trim needed |

> The -V and +V terminals are paralleled pairs — use both screw points for multiple wire runs.

---

### Motor: Metal DC Geared Motor w/Encoder (12V 83RPM)

| Pin  | Description            | Voltage | Connection |
|------|------------------------|---------|------------|
| V+   | Motor power positive   | 12V     | L298 output |
| V-   | Motor power negative   | 12V     | L298 output |
| Vcc  | Encoder supply         | 5V      | Step-down +5V |
| GND  | Encoder ground         | 0V      | System GND |
| VA   | Encoder channel A      | 5V      | Voltage divider → ESP32 GPIO |
| VB   | Encoder channel B      | 5V      | Voltage divider → ESP32 GPIO |

**Motor Specs:**
- Gear ratio: 131:1
- No-load speed: 83 RPM ±10%
- No-load current: 350 mA
- Stall torque: 45 Kg·cm
- Stall current: 7 A
- Encoder: Hall, 16 CPR (motor shaft) / 2096 CPR (gearbox shaft)
- Encoder supply: 5V

---

### Encoder Voltage Divider (×4 signals — VA/VB per motor)

Encoder outputs 5V logic; ESP32 GPIO is 3.3V max. A resistor divider is required on every encoder signal line.

```
VA or VB (5V) ──── 10kΩ ──┬──→ ESP32 GPIO
                           │
                          20kΩ
                           │
                          GND

Output = 5V × (20k / 30k) = 3.33V ✓
```

> 8 resistors total (4× 10kΩ, 4× 20kΩ). Use values from M3 hardware kit or source separately.  
> **Note:** 10kΩ is the top (series) resistor; 20kΩ is the bottom (shunt) resistor. Do not swap.

---

### L298 Dual H-Bridge Motor Driver

| L298 Pin | Connection         | Notes |
|----------|--------------------|-------|
| V+ 1/2   | Motor V+ (Az / El) | 12V motor power |
| V- 1/2   | Motor V- (Az / El) | Return to L298, not GND directly |
| Vcc 1/2  | 5V                 | Logic supply from step-down |
| ENA_1    | ESP32 GPIO 25 (PWM)| Remove ENA jumper if present |
| ENA_2    | ESP32 GPIO 32 (PWM)| Remove ENA jumper if present |
| IN1      | ESP32 GPIO 26      | Az direction |
| IN2      | ESP32 GPIO 27      | Az direction |
| IN3      | ESP32 GPIO 18      | El direction |
| IN4      | ESP32 GPIO 19      | El direction |
| GND 1/2  | System GND         | |
| +12V     | PSU +V             | |
| GND_IN   | PSU -V             | |

> **Important:** Most L298 breakout boards ship with ENA/ENB jumpers populated (ENA tied HIGH = full speed always). Remove these jumpers to enable PWM speed control via ENA pins.

> **L298 voltage drop:** ~2V drop across the H-bridge. Motors see ~10V at 12V supply. Acceptable at this gear ratio.

---

### Step-Down Converter (12V → 5V)

| Pin    | Connection         |
|--------|--------------------|
| +12V   | PSU +V             |
| GND    | System GND         |
| +5V    | ESP32 Vin, L298 Vcc, Motor encoder Vcc |
| GND    | System GND         |

---

### ESP32-WROOM-32UE

| Pin      | Function              | Connected To |
|----------|-----------------------|--------------|
| Vin      | 5V input              | Step-down +5V |
| GND      | Ground                | System GND |
| 3.3V     | 3.3V output           | (do not load heavily) |
| GPIO 25  | Az ENA (PWM)          | L298 ENA_1 |
| GPIO 26  | Az IN1                | L298 IN1 |
| GPIO 27  | Az IN2                | L298 IN2 |
| GPIO 34  | Az VA (encoder, input only) | Voltage divider output |
| GPIO 35  | Az VB (encoder, input only) | Voltage divider output |
| GPIO 32  | El ENA (PWM)          | L298 ENA_2 |
| GPIO 18  | El IN1                | L298 IN3 |
| GPIO 19  | El IN2                | L298 IN4 |
| GPIO 36  | El VA (encoder, input only) | Voltage divider output |
| GPIO 39  | El VB (encoder, input only) | Voltage divider output |

> GPIO 34, 35, 36, 39 are input-only pins on the WROOM-32 — ideal for encoder signals.  
> GPIO 25 and 32 support LEDC PWM hardware channels.

---

## Pin Assignments

```cpp
// Azimuth Motor
#define AZ_ENA   25   // PWM speed (LEDC channel 0)
#define AZ_IN1   26   // Direction
#define AZ_IN2   27   // Direction
#define AZ_VA    34   // Encoder A (input only, interrupt)
#define AZ_VB    35   // Encoder B (input only, interrupt)

// Elevation Motor
#define EL_ENA   32   // PWM speed (LEDC channel 1)
#define EL_IN1   18   // Direction
#define EL_IN2   19   // Direction
#define EL_VA    36   // Encoder A (input only, interrupt)
#define EL_VB    39   // Encoder B (input only, interrupt)
```

---

## Wiring Notes

- **Common GND:** ESP32, L298, step-down converter, and motor encoder GND must all share a common ground rail.
- **Motor decoupling:** Place a 100nF ceramic capacitor across each motor's terminal pair, as close to the L298 output as possible. Prevents motor switching noise from causing ESP32 resets under load.
- **Encoder 5V:** Motor encoder Vcc must be supplied from the 5V step-down rail, not the motor 12V rail.
- **Voltage dividers:** 10kΩ series, 20kΩ shunt per encoder signal. Do not swap resistor positions.
- **PSU chassis ground:** Connect PSU earth terminal (⏚) to enclosure chassis only, not to circuit GND, unless intentionally grounding the system.

---

## Firmware Architecture

**Platform:** PlatformIO, Arduino framework for ESP32

### File Structure

```
minipos/
├── src/
│   ├── main.cpp          # Init and main loop
│   ├── motor.h/.cpp      # PWM and direction control
│   ├── encoder.h/.cpp    # Quadrature decode, tick→degree conversion
│   ├── pid.h/.cpp        # Generic PID controller
│   ├── rotator.h/.cpp    # Az/El position tracking and command handling
│   └── comms.h/.cpp      # Bluetooth SPP, ROT2Prog parser
├── platformio.ini
└── README.md
```

### Module Responsibilities

**motor.cpp**
- Set PWM duty on ENA via LEDC peripheral
- Set IN1/IN2 for direction
- Stop and brake functions

**encoder.cpp**
- Attach interrupts to VA/VB for both axes
- Quadrature decode → signed tick count
- Convert ticks to degrees using gear ratio and CPR

**pid.cpp**
- Generic PID class, instantiated separately for Az and El
- Tunable Kp, Ki, Kd
- Output clamped to PWM range (0–255)

**rotator.cpp**
- Holds current and target Az/El
- Calls PID loop, drives motors
- Enforces position limits before accepting commands

**comms.cpp**
- Bluetooth SPP initialization
- ROT2Prog protocol parser
- Position query response (returns encoder-tracked position)

**main.cpp**
- Initializes all modules
- Main loop calls rotator update at fixed interval (~50 Hz)
- Passes parsed commands from comms to rotator

### Key Constants

```cpp
// Encoder
#define ENCODER_CPR      2096             // counts/rev at gearbox shaft
#define BELT_RATIO       6.667            // pulley reduction
#define TOTAL_CPR        (ENCODER_CPR * BELT_RATIO)  // ticks per output rev
#define TICKS_PER_DEG    (TOTAL_CPR / 360.0)

// PWM
#define PWM_FREQ         5000             // Hz
#define PWM_RESOLUTION   8                // bits (0–255)

// Position Limits
#define AZ_MIN          -360.0
#define AZ_MAX           360.0
#define EL_MIN           0.0
#define EL_MAX           180.0
```

### Command Validation

```cpp
bool setTarget(float az, float el) {
    if (az < AZ_MIN || az > AZ_MAX) return false;
    if (el < EL_MIN || el > EL_MAX) return false;
    target_az = az;
    target_el = el;
    return true;
}
```

---

## Software Stack

```
GPredict  →  Hamlib (rotctld)  →  Bluetooth SPP (virtual COM)  →  ESP32  →  L298  →  Motors
```

**GPredict** handles TLE propagation, pass prediction, and Doppler correction. It outputs Az/El commands to Hamlib.

**Hamlib rotctld** uses the `rot2` backend, which implements the ROT2Prog protocol. It communicates with the ESP32 over a Bluetooth SPP virtual COM port — the host PC sees it as a standard serial port, no custom drivers needed.

**ROT2Prog command set (implement on ESP32):**

| Command | Description |
|---------|-------------|
| `W AZxxx.x ELyyy.y ` | Move to Az/El position |
| `S` | Stop |
| `C2` | Query current position |

---

## Operating Procedure

### First Power-On / Homing

> MiniPos has no endstops. Position is tracked relative to wherever the rotator is pointing at power-on. **Manual homing is required before each tracking session.**

1. Before powering on, physically point the antenna to the home position (Az 0°, El 0° — typically north horizon).
2. Power on the system. ESP32 will initialize with position (0°, 0°).
3. Connect to the ESP32 Bluetooth device from the host PC. Note the assigned COM port.
4. Launch `rotctld` with the rot2 backend pointed at that COM port.
5. Open GPredict and configure the rotator interface to use rotctld.
6. Begin tracking.

### Normal Operation

- GPredict sends Az/El commands during a satellite pass.
- The ESP32 PID loops drive the motors to match commanded position.
- Position queries return encoder-tracked position, not commanded position.
- At the end of a pass, manually return to home if needed before the next session.

---

## Known Limitations & Future Work

| Item | Notes |
|------|-------|
| Manual homing only | No endstops or limit switches. Operator must home before each session. Position resets on power cycle. |
| No wrap-around logic | If GPredict commands a transition across 0° (e.g. 359° → 1°), the rotator takes the long way around. Shortest-path Az logic is a future improvement. |
| ±360° total rotation limit | Software enforces Az within [-360°, +360°]. Commands outside this range are rejected. Physical cable wrap must be managed by the operator. |
| L298 stall current | L298 rated to ~2A continuous. Motor stalls at 7A. Avoid commanding into hard mechanical stops. Consider upgrading to DRV8874 or IBT-2 for more robust current handling. |
| No Doppler compensation | Doppler correction for RF is handled upstream by GPredict/SDR software, not by this rotator controller. |
| Endstop stubs | Endstop support is not implemented but should be added before unattended operation. |
