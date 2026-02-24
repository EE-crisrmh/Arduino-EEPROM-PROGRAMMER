# Architecture Revision — EEPROM Programmer

## Core Change
Migration from Arduino Nano control → dedicated ATmega328P system.

---

## Why
Previous prototype limitations:
- Pin contention
- Hard to scale for PCB
- Control timing tied to Arduino board layout

New design:
- Deterministic pin mapping
- ISP programmable
- Internal clock → fewer components
- PCB-ready architecture

---

## Address Generation Strategy
Unchanged electrically, but now MCU-native:

74HC164 ×2 → 16-bit serial address loader

Advantages:
- Minimal MCU pin usage
- Deterministic update timing
- Direct PCB routing to EEPROM address bus

---

## Control Signal Strategy

### Shift Registers
- SR_DATA (PD2)
- SR_CLK  (PD3)
- SR_CLR  (PD4)

### Reserved for ISP (do not use for GPIO)
- PB3 MOSI
- PB4 MISO
- PB5 SCK

---

## Power Strategy
External regulated +5V input:
- Bulk capacitor at entry
- Local decoupling per IC
- Power LED for bring-up verification

---

## Reset Strategy
- Hardware pull-up
- Manual pushbutton
- ISP-controlled reset

---

## Debug Strategy
Test points for:
- SPI programming
- Shift register control lines
- Power rails

Status LED on PB0 for firmware validation.

---

## Clock Strategy
Internal oscillator:
- No crystal
- Fewer parts
- Purpose-built for programmer firmware

---

## Result
Modular, testable, PCB-safe control platform for EEPROM interface.
