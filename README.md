# Smart Collision Detection & Automatic Emergency Braking (AEB) System

A low-cost, multi-node **Autonomous Emergency Braking** system built on a distributed **CAN bus** architecture, reproducing core architectural patterns found in production automotive ADAS — distributed ECUs, multi-sensor fusion, a hardware fail-safe, event data recording, and (in-progress) adaptive cruise control — using nothing but Arduino and STM32 development boards.

Built at **KPIT APEX Labs**, COEP Technological University.

> Mentor: Jayanth Subramanian, KPIT Technologies

---

## Overview

The system integrates **four independent embedded nodes** on a shared 500 kbps CAN bus:

| Node | Role |
|---|---|
| **Arduino Nano** | Reads joystick input, transmits over 2.4 GHz nRF24L01 RF link |
| **Arduino Uno** | RF → CAN bridge; re-publishes joystick commands onto the CAN bus |
| **Arduino Mega 2560** | Perception ECU — LIDAR + ultrasonic sensor fusion, AEB telemetry, CAN heartbeat origin, hosts the Event Data Recorder |
| **STM32 Nucleo-64 (F446RE)** | Motor controller — arbitrates driver input vs. AEB override, drives the DC motors via an L298N H-bridge, enforces the heartbeat fail-safe |

Rather than simulating these automotive patterns in isolation, the goal was **systems integration**: making sensor fusion, a multi-node CAN network, a hardware-independent fail-safe, event logging, and closed-loop control all interoperate on a single low-cost platform.

## System Architecture

```
Arduino Nano  --2.4GHz RF-->  Arduino Uno  ----0x100---->
                                                          \
Arduino Mega 2560  ----0x200 (telemetry) + 0x050 (heartbeat)---->  STM32 Nucleo F446RE
   (LIDAR + HC-SR04 fusion)                                          (Motor Controller)
                                                                            |
                                                                     L298N H-Bridge
                                                                            |
                                                                       DC Motors
```

### CAN Identifier Allocation

| CAN ID | Source | Purpose |
|---|---|---|
| `0x100` | Arduino Uno (Bridge) | Joystick throttle/steering command |
| `0x050` | Arduino Mega (Perception) | Heartbeat — deadman-switch fail-safe |
| `0x200` | Arduino Mega (Perception) | Fused obstacle distance + AEB zone telemetry |

The STM32 consumes all three IDs and originates no CAN traffic of its own in the current firmware.

## Key Features

- **Zone-based dual-sensor fusion** — LIDAR-Lite v3HP + HC-SR04 ultrasonic readings are cross-validated and combined with distance-dependent weighting (25% / 60% / 90% LIDAR weight by zone), with a moving-average pre-filter and 32-bit-safe fusion math.
- **Multi-stage AEB braking** — CRUISE / DECEL / EMERGENCY zones, with active back-EMF braking in the emergency zone.
- **CAN heartbeat deadman-switch fail-safe** — the motor controller cuts drive output if no heartbeat is received within an 80 ms window, independent of the AEB logic itself. Recovery requires a deliberate, debounced physical button press (no silent auto-recovery).
- **EEPROM-based Event Data Recorder (EDR)** — captures a 21-frame (10 pre / 1 trigger / 10 post) telemetry window at 20 Hz around any emergency-zone trigger, in the spirit of an automotive black box, with heartbeat-preserving writes so the EEPROM flush never trips the watchdog.
- **UDS-inspired diagnostic layer** *(design target)* — request/response state inspection modeled on ISO 14229.
- **PID-based Adaptive Cruise Control** *(design target)* — continuous 300 mm following-gap regulation to replace the discrete DECEL zone.

## Hardware

| Component | Notes |
|---|---|
| Arduino Nano | RF joystick transmitter |
| Arduino Uno | RF → CAN bridge |
| Arduino Mega 2560 | Perception ECU + EDR |
| STM32 Nucleo-64 F446RE | AEB + RC motor controller |
| MCP2515 (×3) | CAN controllers, 500 kbps, SPI |
| LIDAR-Lite v3HP | Primary distance sensor (I2C) |
| HC-SR04 | Secondary ultrasonic sensor |
| nRF24L01 (×2) | 2.4 GHz wireless joystick link |
| L298N | Dual H-bridge motor driver |
| HW504 | Analog joystick |

Full wiring tables, pin assignments, and power distribution are documented in the project appendix (see `/docs`).

## Repository Structure

```
├── firmware/
│   ├── nano_transmitter/       # RF joystick TX
│   ├── uno_bridge/             # RF-to-CAN bridge
│   ├── mega_perception_ecu/    # Sensor fusion + EDR
│   └── stm32_motor_controller/ # AEB zones + fail-safe + motor drive
├── docs/
│   ├── wiring/                 # Per-node wiring tables
│   ├── can-bus/                # CAN ID map, bus load analysis
│   └── test-procedures/        # Watchdog/lockout validation tests
└── README.md
```

## Results (Qualitative Bench Validation)

- Clean CAN frame routing across all four nodes with no observed cross-talk between `0x100`, `0x050`, and `0x200`.
- Correct AEB zone transitions (CRUISE → DECEL → EMERGENCY) as fused distance varied.
- Reliable heartbeat lockout on CAN-H severance / node reset, recoverable only via the physical button.
- Consistent 21-frame EDR capture on every emergency-zone trigger, retrievable over serial.

Formal instrumented timing measurement (CAN analyzer, repeated-trial latency logging) is planned as the next step — see [Limitations](#limitations--future-work).

## Limitations & Future Work

- Sensor suite (LIDAR-Lite v3HP + HC-SR04) is single-point, not full-scene perception.
- No formal CAN bus timing/loading analysis has been run yet (bus load is currently estimated at ~1.1% against a 40% budget).
- ACC currently ships as a linear PWM interpolation ramp; the full PID control law (Section VI of the design paper) is a documented target not yet reflected in firmware.
- UDS-style diagnostics are a design target, not yet implemented in firmware.
- Fault coverage is limited to communication/node-liveness; sensor plausibility checking and CAN bus-off recovery are not yet handled.

## Safety Note

The current bench prototype supplies the STM32's VDD pin from a bench supply measured at ~4.2 V, which exceeds the F446RE's absolute maximum VDD rating (4.0 V). This is a known, unresolved hardware risk on the as-built unit — see `/docs` for details before replicating the power wiring.

## References

- ISO 11898-1:2015 — Road vehicles, Controller Area Network (CAN), Part 1
- ISO 14229-1:2020 — Road vehicles, Unified Diagnostic Services (UDS)
- Microchip MCP2515 Datasheet
- Garmin LIDAR-Lite v3HP Operating Manual
- STMicroelectronics RM0390 / UM1724

## Author

**Meet Shinde**, with Siddhant Wankhede and Vedant Yadav — KPIT APEX Labs, COEP Technological University.
