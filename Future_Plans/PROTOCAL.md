# Serial Protocol (Draft)

## Transport
- USB serial
- Default: 115200 baud (can be changed)
- Newline-delimited ASCII headers + optional binary payload
- All multi-byte numbers are HEX in ASCII headers

## Responses
- OK <message>
- `ERR <code> <message>`
- `DATA <len>` followed by raw bytes
- `ACK <seq>`
- `NAK <seq> <reason>`

## Commands

### INFO
Host:
- `INFO\n`
Device:
- `OK PROTO=1 FW=<ver> CHIP=28C256 SIZE=32768 CHUNK=256\n`

### WRITE (chunked)
Host:
- `WRITE <addr> <len> <seq> <crc32>\n`
- then `<len>` raw bytes
Device:
- `ACK <seq>\n` or `NAK <seq> <reason>\n`

### READ
Host:
- `READ <addr> <len>\n`
Device:
- `DATA <len>\n` then `<len>` raw bytes then `OK\n`

### VERIFY (optional shortcut)
Host:
- `VERIFY <addr> <len> <seq> <crc32>\n`
- then `<len>` raw bytes
Device:
- `ACK <seq>\n` or `ERR MISMATCH <addr> EXP=<xx> GOT=<yy>\n`

## Notes
- Chunk size default: 256 bytes
- Host should retry NAK/timeouts up to N times, then fail
- Firmware must reject out-of-range addr/len


