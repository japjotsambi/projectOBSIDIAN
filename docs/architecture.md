# OBSIDIAN — Architecture

This document describes how OBSIDIAN is structured, how responsibilities are split between the microcontroller (MCU) and the FPGA, and how data flows through the system during each operation.

---

## Overview

OBSIDIAN is built around a single security principle: **the host can use secret keys but can never read them.** All post-quantum cryptography runs on the MCU, but secret key material is stored only inside the FPGA, behind access-control logic. If the host firmware is compromised, an attacker still cannot extract secret keys — and the keys can be destroyed instantly through zeroization.

```
┌─────────────────────────────────┐        SPI         ┌─────────────────────────────────┐
│         MICROCONTROLLER         │◄──────────────────►│              FPGA               │
│          (STM32 / RP2040)       │                    │      (Hardware Key Vault)       │
│                                 │                    │                                 │
│  ┌───────────────────────────┐  │                    │  ┌───────────────────────────┐  │
│  │      ML-KEM Library       │  │                    │  │       SPI Slave           │  │
│  │   (pqm4 / liboqs)         │  │                    │  │                           │  │
│  └───────────────────────────┘  │                    │  └───────────────────────────┘  │
│              │                  │                    │              │                  │
│              ▼                  │                    │              ▼                  │
│  ┌───────────────────────────┐  │   Commands         │  ┌───────────────────────────┐  │
│  │       Key Manager         │──┼──────────────────►─┼──│    Register Interface     │  │
│  │                           │◄─┼───────────────────◄┼──│                           │  │
│  └───────────────────────────┘  │   Responses        │  └───────────────────────────┘  │
│              │                  │                    │              │                  │
│              ▼                  │                    │              ▼                  │
│  ┌───────────────────────────┐  │                    │  ┌───────────────────────────┐  │
│  │       SPI Driver          │  │                    │  │   Access Control Logic    │  │
│  └───────────────────────────┘  │                    │  └───────────────────────────┘  │
│              │                  │                    │              │                  │
│              ▼                  │                    │              ▼                  │
│  ┌───────────────────────────┐  │                    │  ┌───────────────────────────┐  │
│  │        UART CLI           │  │                    │  │   Protected Key Storage   │  │
│  └───────────────────────────┘  │                    │  └───────────────────────────┘  │
└─────────────────────────────────┘                    └─────────────────────────────────┘
         │
         │ UART
         ▼
   ┌───────────┐
   │ Terminal  │
   └───────────┘
```

---

## The security boundary

The SPI bus is the trust boundary. Everything on the MCU side is considered potentially exposed; everything on the FPGA side is the protected domain.

- The MCU may issue commands to **store**, **use**, and **destroy** keys.
- The MCU may **never** read back secret key material unless the FPGA is explicitly unlocked, and even then only within a session that auto-locks.
- The FPGA enforces this in hardware — it is not a software policy that can be patched out.

---

## Component responsibilities

### Microcontroller

| Component | Role |
|-----------|------|
| ML-KEM library | Performs keygen, encapsulation, and decapsulation |
| Key manager | Orchestrates key lifecycle; decides what is stored in hardware vs. used transiently |
| SPI driver | Bare-metal driver that frames commands and transfers bytes to/from the FPGA |
| UART CLI | Interactive command interface for the operator |

### FPGA

| Component | Role |
|-----------|------|
| SPI slave | Receives command bytes, returns response bytes; handles clock-domain crossing |
| Register interface | Decodes commands and addresses; routes reads/writes |
| Access control | Gates secret-key access behind the unlock sequence; runs the auto-lock timer, failed-attempt counter, and zeroization trigger |
| Key storage | Block RAM holding public keys, secret keys, ciphertext, and shared secrets |

---

## Key lifecycle

1. **Generate** — the MCU runs ML-KEM keygen, producing a public/secret keypair in MCU RAM.
2. **Store** — the secret key is written to the FPGA over SPI (`STORE_KEY`).
3. **Wipe host copy** — the MCU securely zeroes its RAM copy of the secret key. From here, the secret exists only in the FPGA.
4. **Use** — for decapsulation, the MCU unlocks the FPGA, loads the secret key into a working buffer, performs the operation, then re-locks.
5. **Destroy** — on `ZEROIZE` (manual or auto), the FPGA overwrites all key storage and sets the zeroized flag.

---

## Data flows

### keygen
```
CLI → Key Manager → ML-KEM keygen (MCU) → secret key → SPI STORE_KEY → FPGA storage
                                          → public key kept in MCU / public buffer
                                          → MCU RAM secret copy wiped
```

### encapsulate
```
CLI (public key) → ML-KEM encaps (MCU) → ciphertext + shared secret → returned to operator
```
Encapsulation needs only a public key, so it requires no unlock.

### decapsulate
```
CLI (ciphertext) → Key Manager → SPI UNLOCK → load secret key → ML-KEM decaps (MCU)
                 → shared secret → SPI LOCK
```
Decapsulation requires the secret key, so it requires an unlock and re-locks afterward.

### zeroize
```
CLI ZEROIZE (confirmed) → SPI ZEROIZE → FPGA multi-pass overwrite of key storage → zeroized flag set
```
Also triggered automatically by the access-control logic after repeated failed unlock attempts.

---

## Tamper response

The access-control block tracks failed unlock attempts. After the configured threshold (default 3), it triggers zeroization automatically, destroying all key material. This converts a brute-force attempt on the unlock code into key destruction rather than eventual access.

---

## Notes & limitations

OBSIDIAN demonstrates HSM **architecture** — the use/read separation, access control, and zeroization. It does not defend against physical side-channel or invasive hardware attacks, and it is not a certified secure element. See the README's Security Model section for the full threat model.
