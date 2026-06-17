# OBSIDIAN

**Operational Bus-Secured Interface for Data Integrity And Nullification**

A hardware-secured key management system that runs post-quantum cryptography (ML-KEM) on a microcontroller while isolating secret keys inside an FPGA, with access control and emergency zeroization. It demonstrates the core architectural principle behind a Hardware Security Module (HSM): the host can *use* keys but can never *read* them.

![Status](https://img.shields.io/badge/status-in%20development-orange)
![Firmware](https://img.shields.io/badge/firmware-C-blue)
![Hardware](https://img.shields.io/badge/hardware-Verilog-purple)
![License](https://img.shields.io/badge/license-MIT-green)

> ⚠️ **Disclaimer:** OBSIDIAN is a personal/educational project for learning embedded security at the hardware–software boundary. It has not been security-audited and is **not intended for production use**. Do not protect real secrets with it.

---

## Features

- **ML-KEM-512** key generation, encapsulation, and decapsulation (post-quantum, NIST FIPS 203)
- **Hardware key isolation** — secret keys live in FPGA memory and are never exposed to host firmware
- **Access control** — secret-key reads require an unlock sequence; auto-locks after a timeout
- **Emergency zeroization** — destroy all key material on demand, with a multi-pass overwrite
- **Tamper response** — automatic zeroization after repeated failed unlock attempts
- **Custom SPI protocol** between the MCU and FPGA, with a documented register map
- **UART command-line interface** for interactive operation

---

## Architecture

OBSIDIAN splits responsibilities across two devices connected over SPI:

```
        UART                          SPI
Terminal ───► Microcontroller  ◄───────────────►  FPGA (key vault)
              • ML-KEM ops                         • SPI slave
              • key manager                         • register interface
              • access policy                       • access control
              • SPI driver                          • protected key storage
              • UART CLI                            • zeroization engine
```

The microcontroller performs all ML-KEM computation and orchestration, but secret keys are stored only in the FPGA behind access-control logic. Even if the host firmware is fully compromised, secret key material cannot be read out without the unlock sequence — and can be destroyed instantly via zeroization.

See [`docs/architecture.md`](docs/architecture.md) for the full block diagram and [`docs/protocol.md`](docs/protocol.md) for the complete SPI protocol and register map.

### Why ML-KEM?

ML-KEM is a key encapsulation mechanism — it establishes shared secrets for confidentiality, which is exactly the job of a key-management module. Because it is lattice-based and NIST-standardized, the keys it manages are resistant to future quantum attacks, including the "harvest-now, decrypt-later" threat model.

---

## Hardware Requirements

| Item | Recommendation | Approx. cost |
|------|----------------|--------------|
| Microcontroller | Raspberry Pi Pico (RP2040) or STM32 dev board | ~$4–15 |
| FPGA board | Basys 3 (Artix-7) or equivalent | varies |
| Logic analyzer | Inexpensive 8-channel clone (for SPI debug) | ~$10–15 |
| Jumper wires | Male-to-male / male-to-female | ~$5 |
| USB–UART adapter | Only if not built into the MCU board | ~$5 |

---

## Repository Structure

```
obsidian/
├── fpga/
│   ├── src/            # Verilog: spi_slave, register_interface, access_control, key_storage
│   ├── sim/            # Testbenches
│   └── constraints/    # Board pin constraints (.xdc)
├── firmware/
│   ├── src/            # SPI driver, key manager, CLI, main
│   ├── include/        # Headers
│   └── lib/            # ML-KEM library (e.g. pqm4)
├── docs/
│   ├── architecture.md
│   └── protocol.md
└── README.md
```

---

## Getting Started

### Prerequisites

- **FPGA:** Xilinx Vivado (WebPACK edition is free)
- **Firmware:** the toolchain for your MCU — Pico SDK + CMake (RP2040) or arm-none-eabi-gcc / STM32CubeIDE (STM32)
- A serial terminal (e.g. `minicom`, `screen`, PuTTY) at the configured baud rate

### Build the FPGA

```bash
cd fpga
# Open the project in Vivado, or build from the TCL flow:
vivado -mode batch -source build.tcl
# Program the bitstream to your board
```

### Build the firmware

```bash
cd firmware
mkdir build && cd build
cmake ..          # adjust for your MCU toolchain
make
# Flash the resulting binary to your MCU
```

### Wiring (example pin map)

| MCU | FPGA header | Signal |
|-----|-------------|--------|
| SCLK | JA1 | SPI_CLK |
| MISO | JA2 | SPI_MISO |
| MOSI | JA3 | SPI_MOSI |
| CS   | JA4 | SPI_CS |
| GND  | GND | Ground |

Confirm both sides agree on SPI mode (clock polarity/phase) before debugging anything else.

---

## Usage

Connect to the MCU over UART and you'll get the OBSIDIAN prompt.

| Command | Description |
|---------|-------------|
| `keygen [id]` | Generate an ML-KEM keypair and store the secret in hardware |
| `pubkey [id]` | Print a stored public key (hex) |
| `encaps [id] [pk_hex]` | Encapsulate against a public key |
| `decaps [id] [ct_hex]` | Decapsulate a ciphertext (requires unlock) |
| `status` | Show key slots, lock state, and zeroization status |
| `lock` | Lock the FPGA |
| `zeroize` | Destroy all keys (requires confirmation) |
| `help` | List commands |

### Example session

```
OBSIDIAN Secure Key Manager v1.0
Type 'help' for commands.

OBSIDIAN> keygen 0
Generating ML-KEM-512 keypair...
Storing secret key to hardware...
Done. Key ID 0 ready.

OBSIDIAN> encaps 0 7a3f8c2b...
Ciphertext:    9d4e...
Shared secret: a1b2c3d4...

OBSIDIAN> status
Key slots:
  [0] PUBLIC: yes, SECRET: hardware
  [1] empty
FPGA: locked
Failed unlock attempts: 0/3
Zeroized: no

OBSIDIAN> zeroize
WARNING: This will DESTROY ALL KEYS permanently.
Type 'CONFIRM' to proceed: CONFIRM
Zeroizing...
[████████████████████████] 100%
ALL KEYS DESTROYED.
```

---

## Security Model

**What it protects against**
- A compromised host that tries to read secret keys directly — secret-key reads are gated behind the FPGA unlock sequence
- Brute-forcing the unlock code — repeated failures trigger automatic zeroization
- Recovery of keys after destruction — zeroization overwrites key storage in multiple passes

**What it does *not* protect against**
- Physical side-channel attacks (power/EM analysis) on the FPGA or MCU
- Invasive hardware attacks (decapping, probing internal memory)
- A host that is authorized and unlocked but malicious during a session

This is an architectural demonstration of HSM key-isolation principles, not a hardened or certified secure element.

---

## Performance

Measured on `<MCU / clock speed>` — to be filled in.

| Operation | Time |
|-----------|------|
| keygen | TBD |
| encaps | TBD |
| decaps | TBD |

---

## Roadmap / Status

- [ ] FPGA SPI slave + register interface
- [ ] Protected key storage + access control
- [ ] Zeroization engine
- [ ] MCU bare-metal SPI driver
- [ ] ML-KEM integration
- [ ] Key manager + MCU↔FPGA integration
- [ ] UART CLI
- [ ] End-to-end integration + testing
- [ ] Security hardening
- [ ] Documentation + demo

---

## References

- NIST FIPS 203 — ML-KEM (Module-Lattice Key-Encapsulation Mechanism)
- [pqm4](https://github.com/mupq/pqm4) — post-quantum crypto for ARM Cortex-M
- [liboqs](https://github.com/open-quantum-safe/liboqs) — Open Quantum Safe library
- [Nandland SPI Slave in Verilog](https://nandland.com/spi-slave-verilog/)

---

## License

Released under the MIT License — see [`LICENSE`](LICENSE). (Swap in a different license if you prefer.)

---

## Acknowledgements

Built as a personal project to explore post-quantum cryptography and secure key management at the hardware–software boundary.
