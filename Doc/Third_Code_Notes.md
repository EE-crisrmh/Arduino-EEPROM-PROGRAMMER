# Notes — EEPROM Programmer Prototype Debug Log

## Date
2026-02-19

## Goal of today
Prove whether the prototype can do reliable read + write after adding proper /OE and /CE control (bus contention fix). If it still fails, determine whether the failure is data bus, control lines, or address shifting.

---

## What I changed today (wiring + firmware intent)

### Control pins (active-low)
- Moved EEPROM control from “tied low” to Arduino-controlled:
  - /WE on D12
  - /CE on D5
  - /OE on D6
- Firmware change intent:
  - During writes: disable /OE first so EEPROM never drives the bus while Arduino is driving (prevents contention).
  - During reads: set data pins to INPUT, enable /CE and /OE.

### Data bus mapping (due to pin availability + dead EEPROM bit)
- EEPROM D0..D2 → Arduino A0..A2
- EEPROM D4..D7 → Arduino D8..D11
- EEPROM D3 (bit 3) is physically broken → always mask bit 3 in write/read/compare

---

## Tests run today, why they were run, and what they proved

### Test 1 — “Why do I read the same byte everywhere?”
**Observation:** reads returned `0x77` for many addresses (ex: R 0000, R 0001, R 00FF all returned 77).

**Why run this test:** If the value is constant across different addresses, either:
- the address is not changing (address shifter chain dead), OR
- the data bus is stuck/floating and not actually reading EEPROM output.

**What it proved:** By itself, constant reads do not prove which subsystem is broken. It triggered the next tests.

---

### Test 2 — DMM check of EEPROM data pins during read
Measured at EEPROM pins while a read was happening:
- D0 ≈ 0.227 V
- D4 ≈ 0.239 V
- D7 ≈ 0.204 V

**Why run this test:** Compare “what firmware thinks it reads” vs actual physical voltages at EEPROM pins.

**What it proved:** The physical data pins looked LOW while firmware still reported `0x77`, which strongly suggested either:
- the bit-to-pin mapping is wrong somewhere, OR
- the bus is not being read from the EEPROM pins we think it is.

---

### Test 3 — Bus “float vs driven” test (EEPROM disabled vs enabled)
A dedicated sketch printed bus values in two states:

- EEPROM disabled: `/CE=HIGH` and `/OE=HIGH` → bus should float  
- EEPROM enabled: `/CE=LOW` and `/OE=LOW` → EEPROM should drive bus

**Result example:**
- `CE=H OE=H 36   CE=L OE=L 77`

**Why run this test:** It isolates whether the EEPROM is influencing the bus at all.

**What it proved:**
- The bus changes between disabled vs enabled → EEPROM output enable path is doing something.
- But the enabled value being constant across addresses still points to address lines not changing.

---

### Test 4 — Address toggle test (read 0x0000 vs 0x1234 repeatedly)
A test sketch repeatedly read two different addresses:

- Expected: values should differ most of the time if addresses are changing.
- Actual: `0000=77  1234=77` repeatedly.

**Why run this test:** It directly checks whether the address shifter chain is changing the EEPROM address.

**What it proved:** Address lines were not changing at the EEPROM (or not reaching it). This is an address path problem, not a write-timing problem.

---

### Test 5 — 74HC164 single-chip LED shift test
Built a minimal test:
- One 74HC164 only
- /MR tied to +5V
- CP clocked from Arduino
- Serial data from Arduino
- LED on Q0

**Observation:** LED blinked as expected.

**Why run this test:** Prove that the Arduino clock/data pins and at least one shift register are functional.

**What it proved:** The base shift register logic works (chip 1 is alive and shifts correctly).

---

### Test 6 — 74HC164 two-chip chained LED shift test (16-bit)
Chained the second 74HC164:
- Chip 2 inputs driven from chip 1 Q7
- Same clock to both
- /MR tied high for both
- LED on chip 2 output

**Observation:** LED blinked on chip 2 as expected.

**Why run this test:** Prove the 16-bit shift chain (chip1 → chip2) actually propagates data.

**What it proved:** The shift chain itself can work in isolation.

---

## Current conclusion (end of session)
- Data bus control (/OE, /CE) is conceptually correct and bus state changes when enabling/disabling the EEPROM.
- 74HC164 chain can shift data (verified with LED tests).
- Reads returning `0x77` at all addresses strongly implies: **EEPROM address pins are not being driven by the shift register outputs, or not wired correctly to the EEPROM address pins.**
- The remaining work is hardware integration: connecting 74HC164 Q outputs to EEPROM A0–A14 correctly and verifying the address lines change at the EEPROM pins.

---

## Next session test plan (what to test next)

### Priority 1 — prove address pins change at the EEPROM
Run address toggle (0x0000 vs 0x0001, then 0x0000 vs 0x7FFF) and measure directly at EEPROM pins:
- A0 should toggle when switching 0x0000 ↔ 0x0001
- A1 should stay constant in that same test
- For 0x0000 ↔ 0x7FFF, multiple address pins should change

If A0–A14 don’t change at the EEPROM pins, fix the wiring between shift-register outputs and EEPROM address inputs.

### Priority 2 — verify /CE and /OE at the EEPROM pins
Measure voltages directly at the EEPROM pins during read:
- /CE near 0V
- /OE near 0V
If they’re not, wiring to EEPROM control pins is wrong.

### Priority 3 — re-run address toggle read test
Once A-lines are confirmed changing at EEPROM pins:
- rerun `0000` vs `1234` read test; values should differ.
- rerun float-vs-driven test; driven value should depend on address.

### Priority 4 — restore write/verify
Only after reads vary with address:
- rerun spot test
- then full write + verify
- track bit error counts (excluding dead bit 3)

---

## Known risks / things left to test
- Incorrect mapping of 74HC164 outputs (Q0..Q7) to EEPROM A0..A7 / A8..A14
- Breadboard row/rail breaks causing address lines not actually connected
- Wrong assumption about which physical pins are Q7 / A0 etc (pinout mismatch)
- EEPROM /CE and /OE not actually reaching EEPROM pins (measured at wrong node previously)
