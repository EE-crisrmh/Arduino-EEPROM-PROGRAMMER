# Software Tasks (Host + Firmware)

This document describes what a software contributor needs to implement to make the programmer feel like a real tool on macOS.

## Required skills
- Comfortable in terminal
- Python experience
- Serial communication (USB CDC / `/dev/cu.*`)
- Handling binary files (`.bin`)
- Basic error handling and packaging

---

## 1) Define and implement a robust serial protocol
### Goal
Host and Arduino communicate with clear commands, responses, checksums, and retries.

### Requirements
- Versioned protocol (`PROTO 1`)
- Chunked transfers (e.g., 256 bytes per chunk)
- Per-chunk checksum (CRC32 or simple checksum)
- Explicit ACK/NAK responses
- Timeouts + retry strategy
- Useful error codes/messages

Deliverables:
- `future-plans/PROTOCOL.md` finalized
- Firmware implements the protocol
- Host tool uses the protocol

---

## 2) Build a macOS CLI tool
### Commands
- `eeprom info`
  - prints device + firmware version + supported chip geometry
- `eeprom flash <file.bin> [--verify]`
  - writes file to EEPROM, optional verify (default verify recommended)
- `eeprom verify <file.bin>`
  - compares EEPROM contents to file
- `eeprom read <out.bin>`
  - dumps EEPROM contents to a file

### CLI requirements
- Auto-detect serial port (or allow `--port`)
- Progress bar (bytes written / total)
- Clear failures (first mismatch address, expected vs actual)
- Exit codes suitable for automation

Deliverables:
- `host/` folder with CLI implementation
- `README` usage examples
- Basic unit tests for file/protocol parsing

---

## 3) Firmware improvements for reliability
### Requirements
- Input validation (address range, length)
- Deterministic state machine (no “random blocking waits”)
- Configurable timing constants for AT28C256
- Optional “verify while writing” mode

Deliverables:
- Firmware refactor for clean command handling
- Logging hooks (optional)

---

## 4) Packaging / distribution (macOS)
Pick one:
- Python + `pipx` install instructions
- Single-file Python script with minimal deps
- Go single-binary release (best UX)

Deliverables:
- Install instructions
- Tagged release artifacts (optional)
