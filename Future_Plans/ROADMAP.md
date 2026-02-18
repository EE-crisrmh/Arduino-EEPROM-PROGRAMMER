# Roadmap

This project is already proven in hardware: the Arduino-based programmer successfully programs and verifies AT28C256 EEPROMs used in a working 6502 computer.

Next phase focuses on:
1) making the workflow polished and repeatable on macOS
2) migrating the hardware to a PCB for reliability and easier use

---

## Phase 1 — Software polish (macOS-first)

### Goals
- A reliable CLI tool to flash/verify/read EEPROM images over USB-serial
- A stable, versioned serial protocol between host and Arduino
- Clear errors, retries, and progress reporting

### Deliverables
- `host/` CLI tool with commands:
  - `flash <file.bin>`
  - `verify <file.bin>`
  - `read <out.bin>`
  - `info`
- Protocol document + firmware implementing it
- Simple release packaging (macOS)

---

## Phase 2 — PCB version

### Goals
- Replace breadboard/wiring with a manufacturable PCB
- Improve reliability, connectors, labeling, and testability

### Deliverables
- PCB schematic + layout files
- BOM
- Assembly notes + validation checklist

---

## Phase 3 — Quality upgrades 
- Better performance (chunking, smarter verify)
- Expanded chip support (same voltage family)
- Optional GUI wrapper
