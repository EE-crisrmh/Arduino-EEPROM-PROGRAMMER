# Tests I ran (work log)

## Context
A key complication is that EEPROM I/O3 (data bit 3) is physically broken on my current chip. Because of that, I mask out bit 3 (0x08) during comparisons. This lets me validate the other 7 bits even with the damaged pin.

My goal for these tests was to confirm:
1) Address shifting works (the EEPROM address changes when I shift different bits)
2) Data bus mapping is correct (D0–D7 are actually the bits I think they are)
3) Writes actually program the EEPROM (not just “I sent a write command”)
4) Reads are stable and repeatable
5) The test interface works reliably (serial commands / batch commands)

---

## Test 1 — Baseline read stability at a fixed address
### What I did
I read address `0x0000` repeatedly using the serial read command.

Example:
- `R 0000`
- `R 0000`
- `R 0000`

### What I expected
If address shifting and data reads are stable, the same address should return the same byte every time.

### What I observed
I consistently got a stable value when repeating reads (examples seen during testing included `E7` and later `F7`). The exact value itself is not “good” or “bad” by default — the important part is that it was repeatable for the same address.

### What this told me
- Reads are working at least well enough to return stable bytes.
- Address `0x0000` is being reached consistently (or at minimum the same physical location is being reached consistently).

---

## Test 2 — Spot write + immediate verify (address 0x0000)
### What I did
I attempted to write a known value to address `0x0000` and then read it back.

Example intended sequence:
- `W 0000 AA`
- `R 0000`
- `W 0000 55`
- `R 0000`

Also important: because bit 3 is broken, I compare using a mask:
- expected = written_value & ~0x08
- actual   = read_value & ~0x08

### What I expected
If writes are working, readback should match the written value (except for bit 3 which is masked out).

Examples:
- Write `AA` (10101010) → masked expected `A2`
- Write `55` (01010101) → masked expected `55`

### What I observed
I saw readback values change after writes, which suggests write activity was happening. However, I did not get clean readback equal to the expected masked value. I observed inconsistent results such as `E7`, `F7`, and `67` appearing during different attempts.

### What this told me
- The system can acknowledge a write command (`I ok`), but acknowledgement is not proof of a correct write.
- Something in the write/read path is inconsistent: either bus contention, bit mapping mismatch, timing, or address shifting error during the write cycle.

---

## Test 3 — Full “spot test” in firmware (automatic)
### What I did
I ran the firmware’s automatic spot test (it writes to a small set of addresses and verifies them).

The firmware reported:
- `start spot test`
- `spot FAIL 0 exp=0 got=E7`
- `spot test failed`

### What I expected
Spot test should pass if basic write+read works for a few addresses.

### What I observed
Spot test failed immediately at address `0x0000`: expected `0x00` but got `0xE7`.

### What this told me
The write cycle either did not happen or did not produce correct data. The EEPROM returned whatever was already stored at address 0, or I got corrupted output on the bus.

---

## Test 4 — Serial command interface reliability (single commands)
### What I did
I tested that the Arduino is actually receiving and parsing the command lines correctly.

I enabled debug prints so the Arduino would echo what it received:
- `RX: [R 0000]`

Then I typed a single read command:
- `R 0000`

### What I expected
The Arduino should show it received the correct command and then print a 2-digit hex response.

### What I observed
This worked reliably for single commands.
Example:
- `RX: [R 0000]`
- `F7`

### What this told me
The serial interface works when I send commands one at a time.

---

## Test 5 — Batch command paste behavior (multiple commands pasted at once)
### What I did
I tried to paste multiple lines into Arduino IDE Serial Monitor at once (e.g., the walking-1 pattern or multiple W/R pairs).

### What I expected
Each line would be processed as its own command (like a real terminal).

### What I observed
The Serial Monitor did not reliably deliver each line separately. I saw cases where multiple commands were mashed into one buffer string, resulting in debug output like:
- `RX: [0000 80 R 0000]`
followed by:
- `I err`

### What this told me
Arduino IDE Serial Monitor is not a robust batch sender for this workflow. It can merge, drop, or reformat pasted content depending on line endings and timing.

---

## Test 6 — Firmware changes to handle mashed/pasted command strings
### What I did
I updated the firmware so it can “scavenge” commands from a mashed input string (searching for occurrences of `R ` and `W ` within the received chunk) instead of requiring the command to start at the beginning of the line.

### What I expected
I should be able to paste blocks of commands and have the firmware still execute each `R`/`W` embedded in the input.

### What I observed
After this change, I started getting multiple `I ok` acknowledgements and read outputs in one run, instead of only one action or immediate errors.

However, the write/read results were still inconsistent at the EEPROM level (acknowledgements were not the same as correct readback).

### What this told me
The command transport problem (Serial Monitor batching) can be mitigated in firmware.
But the EEPROM programming correctness problem still remains.

---

## Test 7 — Attempted data bus validation via different write patterns
### What I did
I used different write patterns (e.g., `AA`, `55`, and others) and watched the readback values.

### What I expected
Readback should match the pattern (masked).

### What I observed
Readback changed sometimes, but did not cleanly match the expected masked values, suggesting corruption or bit mapping issues.

### What this told me
The system may be writing, but not correctly / not deterministically.

---

# Summary of what I proved today
✅ I can read stable values from at least some addresses (e.g. repeated reads at 0x0000 return the same byte).  
✅ The firmware can parse single commands reliably.  
✅ Batch/paste behavior can be improved with firmware changes.  
⚠️ Write acknowledgements exist, but I cannot claim reliable programming yet.  
❌ Write + verify is inconsistent and fails the automatic spot test.

---

# What I need to fix (for reliability)

## Electrical / architecture fixes (highest priority)
1) **Control OE (and ideally CE) from the Arduino**
   - Right now my control pin strategy can cause the EEPROM to drive the data bus while the Arduino is also driving it during writes.
   - That can cause bus contention and inconsistent programming/readback.
   - On the PCB I need OE on a controllable pin so I can tri-state the EEPROM output during writes.

2) **Data bus direction control discipline**
   - During writes: Arduino drives D0–D7, EEPROM output should be disabled.
   - During reads: Arduino sets D0–D7 as inputs, EEPROM output enabled.

## Firmware fixes
1) **Verify address shifting order is correct (A0–A14 mapping)**
   - Even if reads are stable, I need to confirm I’m truly hitting the intended address lines.
   - I should add a read-only address sweep and compare repeated snapshots for stability.

2) **Improve the write pulse timing**
   - I need to keep WE pulse width within spec and keep address/data stable around the pulse.

3) **Better verification reporting**
   - Print expected vs actual masked bytes.
   - Track which bits mismatch most often to identify wiring errors vs contention.

## Tooling/process fixes
1) **Stop using Arduino IDE Serial Monitor as a batch sender**
   - It merges/drops pasted lines.
   - I should use a real serial terminal or a Python script once the firmware supports stable commands.

2) **Build a dedicated automated tester**
   - Once the hardware control pins are correct, I should automate:
     - walking-1 / walking-0 test
     - checkerboard patterns
     - random pattern test with seeded PRNG
     - full-chip write + verify
