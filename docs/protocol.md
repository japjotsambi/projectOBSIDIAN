# OBSIDIAN — SPI Protocol Specification

This document defines the SPI command/response protocol and register map used between the microcontroller (master) and the FPGA (slave).

---

## Physical layer

| Parameter | Value |
|-----------|-------|
| Role | MCU = SPI controller, FPGA = SPI peripheral |
| Bit order | MSB first |
| Word size | 8 bits |
| Mode | Mode 0 (CPOL = 0, CPHA = 0) by default — confirm both sides agree |
| Chip select | Active low; one full transaction per CS-low window |
| Clock rate | Start low (e.g. 1 MHz) for bring-up; raise after timing is verified |

All multi-byte fields are transferred MSB-first. The FPGA performs clock-domain crossing between the SPI clock and its system clock internally.

---

## Frame formats

### Command frame (MCU → FPGA)

```
┌──────────┬───────────┬───────────────────────┐
│ CMD (1B) │ ADDR (1B) │ DATA (0–N bytes)       │
└──────────┴───────────┴───────────────────────┘
```

- `CMD` — operation opcode (see below)
- `ADDR` — register address or buffer selector the command operates on
- `DATA` — optional payload; length depends on the command

### Response frame (FPGA → MCU)

```
┌──────────┬───────────────────────┐
│ STATUS   │ DATA (0–N bytes)       │
└──────────┴───────────────────────┘
```

- `STATUS` — result code (see below)
- `DATA` — optional returned payload; present only on success for read-type commands

---

## Command opcodes

| Opcode | Name | Payload | Notes |
|--------|------|---------|-------|
| `0x01` | `WRITE_REG` | 1 byte | Write a single control/status register |
| `0x02` | `READ_REG` | — | Read a single control/status register |
| `0x03` | `STORE_KEY` | length + bytes | Block-write key material into a buffer |
| `0x04` | `LOAD_KEY` | length | Block-read key material (requires unlock) |
| `0x05` | `UNLOCK` | 4 bytes | Provide the 32-bit unlock code |
| `0x06` | `LOCK` | — | Re-lock access immediately |
| `0x07` | `ZEROIZE` | — | Destroy all key material |

---

## Status codes

| Code | Name | Meaning |
|------|------|---------|
| `0x00` | `OK` | Command succeeded |
| `0x01` | `ERROR_LOCKED` | Protected resource accessed while locked |
| `0x02` | `ERROR_INVALID_CMD` | Unknown opcode |
| `0x03` | `ERROR_INVALID_ADDR` | Address out of range |

---

## Register map

Single-byte control and status registers are accessed with `READ_REG` / `WRITE_REG`. Larger key buffers are accessed as **blocks** via `STORE_KEY` / `LOAD_KEY` (see below) — the `ADDR` value selects which buffer the block transfer targets.

| Address | Register | Access | Description |
|---------|----------|--------|-------------|
| `0x00` | `STATUS` | R | Device status: ready, busy, error, zeroized |
| `0x01` | `CONTROL` | R/W | Control bits: unlock, lock, zeroize |
| `0x02` | `KEY_ID` | R/W | Selects the active key slot |
| `0x03` | `FAIL_COUNT` | R | Failed unlock attempts (auto-zeroize at 3) |
| `0x10` | `PUBLIC_KEY` | R/W | Public key buffer (block) — read allowed |
| `0x20` | `SECRET_KEY` | R\* | Secret key buffer (block) — \*read requires unlock |
| `0x30` | `CIPHERTEXT` | R/W | Ciphertext buffer (block) |
| `0x40` | `SHARED_SECRET` | R\* | Shared-secret buffer (block) — \*read requires unlock |

\* Reads of `SECRET_KEY` and `SHARED_SECRET` succeed only while the device is unlocked; otherwise the FPGA returns `ERROR_LOCKED`.

### Buffer sizes (ML-KEM-512)

| Buffer | Size |
|--------|------|
| Public key | 800 bytes |
| Secret key | 1632 bytes |
| Ciphertext | 768 bytes |
| Shared secret | 32 bytes |

Because these exceed a single register, they are not byte-addressed individually. Each buffer has a base selector (the addresses above), and the FPGA maintains an internal auto-incrementing offset during a block transfer. A `STORE_KEY` / `LOAD_KEY` transaction carries an explicit length so the FPGA knows how many bytes follow.

---

## Block transfers

### STORE_KEY (MCU → FPGA)

```
CMD=0x03 │ ADDR=buffer │ LEN(2B) │ DATA[LEN] ...
```
The FPGA streams `LEN` bytes into the selected buffer starting at offset 0, then returns a single `STATUS` byte.

### LOAD_KEY (FPGA → MCU)

```
CMD=0x04 │ ADDR=buffer │ LEN(2B)        (then clock out the response)
Response: STATUS │ DATA[LEN] ...
```
If the buffer is access-protected and the device is locked, the FPGA returns `ERROR_LOCKED` with no data.

---

## Example transactions

**Read STATUS**
```
MCU →  0x02 0x00
FPGA ← 0x00 <status_byte>
```

**Write CONTROL (request lock)**
```
MCU →  0x01 0x01 <control_byte>
FPGA ← 0x00
```

**Unlock**
```
MCU →  0x05 0x00 <code[3]> <code[2]> <code[1]> <code[0]>
FPGA ← 0x00            (or ERROR; failed attempts increment FAIL_COUNT)
```

**Store a secret key**
```
MCU →  0x03 0x20 0x06 0x60  <1632 bytes>
FPGA ← 0x00
```

**Decapsulation flow (host orchestration)**
```
1. UNLOCK with the code
2. LOAD_KEY from SECRET_KEY buffer        → secret key into MCU working RAM
3. (MCU performs ML-KEM decaps)
4. LOCK
5. (MCU wipes the working RAM copy)
```

**Zeroize**
```
MCU →  0x07 0x00
FPGA ← 0x00            (key storage overwritten; STATUS now reports zeroized)
```

---

## Access-control behavior

- `SECRET_KEY` and `SHARED_SECRET` reads require a prior successful `UNLOCK`.
- The unlock state auto-clears after a timeout (or on explicit `LOCK`).
- Each failed `UNLOCK` increments `FAIL_COUNT`; reaching the threshold (default 3) triggers automatic `ZEROIZE`.
- After zeroization, all key buffers read back as zero and `STATUS` reports the zeroized state until reset/re-provisioning.
