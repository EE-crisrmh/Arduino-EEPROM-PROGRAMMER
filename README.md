# Arduino AT28C256 EEPROM Programmer (macOS-friendly)

A standalone EEPROM programmer built to support rapid ROM iteration for a 6502 computer without relying on Windows-only tooling.  
This project uses an **Arduino nano** and **2x shift registers** to drive the EEPROM address bus, perform **parallel byte writes**, and run **read-back verification** to ensure data integrity.

This repo contains:
- The **firmware** that runs on the Arduino and directly controls the EEPROM.
- A **host-side tool spec** (macOS) to upload `.bin` images over USB serial, similar to a minimal “TL48 workflow,” but focused on AT28C256.

---

## Why this exists

I purchased a TL48 programmer but the workflow was not reliable on my macOS. Rather than depend on Windows software, I built a dedicated programmer that:
- Works over standard USB serial
- Supports full-chip programming + verify
- Fits the actual needs of a 6502 ROM development loop

This programmer has already been used to successfully program ROMs for a working 6502 computer.

---

## System Architecture

### Hardware (high-level)
- **Arduino Nano** orchestrates all control and timing.
- **Shift registers** generate the full address bus while minimizing GPIO usage.
- **Parallel data bus** is driven directly for byte writes and reads.
- Control signals (WE/OE/CE) are toggled with correct timing for AT28C256 write cycles.

### Firmware responsibilities (Arduino)
The Arduino firmware is responsible for:
- Setting an address (`setAddress(addr)`)
- Writing a byte (`writeByte(addr, value)`)
- Reading a byte (`readByte(addr)`)
- Verifying a block or entire chip (`verify(image)`)

### Host tool responsibilities (macOS)
The host tool is responsible for:
- Loading a ROM image (`.bin`)
- Sending commands/data to the Arduino over USB serial
- Displaying progress + errors
- Optionally doing a second-pass verify for confidence

---

## Features

- **AT28C256 support**
- **Byte-level programming**
- **Read-back verification**
- **Deterministic timing control**
- Designed for a fast ROM iteration loop (edit → build → flash → test)

---

## Host Tool: How a Software Developer can contribute: (More in Future Plans folder) 

The goal is a small CLI tool that feels like:

```bash
eeprom flash firmware.bin
eeprom verify firmware.bin
eeprom read dump.bin
eeprom info
