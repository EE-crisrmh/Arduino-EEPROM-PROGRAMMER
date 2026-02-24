# EEPROM Programmer — Lab Notes

## Date
2026-02-24

---

## Objective
Stabilize the address generation subsystem and migrate the control architecture from Arduino Nano → standalone ATmega328P design for PCB.

---

## What Was Built and Tested

### 1. Breadboard Prototype — 74HC164 Shift Register Chain
- Device: 74HC164 ×2
- Verified functional as a 16-bit shift chain

#### Wiring (per datasheet pinout)
For each 74HC164:
- VCC (14) → +5V
- GND (7) → GND
- CLK (8) → SR_CLK
- CLR (9) → SR_CLR
- A (1) and B (2) tied together

Chain:
- Chip1 QH (13) → Chip2 A+B

Control from MCU:
- SR_DATA → serial data in (chip1 A+B)
- SR_CLK → common clock
- SR_CLR → common clear

Decoupling:
- 0.1 µF per IC

#### Test Method
Firmware patterns:
- Walking 1
- Block fill / flush

#### Result
- Bit propagation confirmed across both ICs
- Visual verification using LEDs on QA outputs
- Chain direction and timing confirmed

---

### 2. ATmega328P Standalone Architecture (Schematic Complete)
- Package: DIP-28
- Programming method: ISP only
- Clock source: internal oscillator (no crystal)

#### Power
- VCC (7), AVCC (20) → +5V
- GND (8), GND (22) → GND
- AREF → 0.1 µF → GND
- 0.1 µF decoupling on VCC and AVCC
- 10 µF bulk on 5V rail

#### Reset
- 10k pull-up to +5V
- Reset pushbutton to GND
- Reset routed to ISP

#### ISP Header (2×3)
- MISO → PB4 (18)
- MOSI → PB3 (17)
- SCK → PB5 (19)
- RESET → pin 1
- VCC, GND

#### Shift Register Control Pins
- SR_DATA → PD2 (pin 4)
- SR_CLK  → PD3 (pin 5)
- SR_CLR  → PD4 (pin 6)

#### Power Input
- External +5V and GND connector
- Bulk cap at entry
- Power LED

#### Debug
- Status LED on PB0

---

## Problems Encountered
- Grid misalignment in KiCad causing false net connectivity
- Hidden power pins in MCU symbol
- Initial clock/data misrouting on shift registers
- Breadboard short during second SR integration → thermal event → corrected by staged reassembly

---

## Verified Working Today
- 16-bit shift chain hardware
- Correct bit propagation
- Clean MCU schematic with ERC passing
- ISP net connectivity

---

## Next Step
Return to breadboard prototype and integrate EEPROM:
- Reintroduce data bus
- Reintroduce /CE /OE /WE control
- Perform address-line validation using proven shift chain
