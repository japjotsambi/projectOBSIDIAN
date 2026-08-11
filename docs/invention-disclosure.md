# Invention Disclosure — Draft

*Confidential internal disclosure document. Prepared as a coursework exercise in invention disclosure and prior-art assessment.*

---

## 1. Title

OBSIDIAN: Hardware-Isolated Post-Quantum Key Management with Emergency Zeroization

## 2. Inventor(s)

- `<Your name>`, Purdue University — Computer Engineering
- Date conceived: `<date>`
- Date first reduced to practice: `<date>`

## 3. Technical field

Embedded security; hardware security modules (HSMs); post-quantum cryptography; FPGA-based key isolation.

## 4. Problem addressed

- On a conventional microcontroller, secret keys live in memory the firmware can read — a compromised host can exfiltrate them.
- Classical key-management hardware is not resistant to future quantum attacks.
- Devices fielded in hostile environments need a way to destroy keys rapidly and verifiably.

## 5. Summary

A two-device key-management system in which a microcontroller (MCU) performs ML-KEM (NIST FIPS 203) operations, but secret keys are stored exclusively inside an FPGA and are reachable only through an access-controlled SPI register interface. The host can request key *use* but cannot *read* secret material without an unlock sequence. The FPGA performs multi-pass zeroization on command and automatically zeroizes after a threshold of failed unlock attempts.

## 6. How it works (brief)

MCU ↔ SPI ↔ FPGA. FPGA modules: SPI slave, register interface, access control (unlock / lock / auto-lock / fail-counter), protected key storage, and a zeroization engine. Full detail in `docs/architecture.md` and `docs/protocol.md`.

## 7. Candidate novel / inventive aspects (to be assessed — not yet claimed)

1. Combining a post-quantum KEM on a host MCU with FPGA-resident secret-key isolation in a low-cost discrete (non-ASIC) design.
2. Tamper response that converts repeated failed authentication into automatic key zeroization in this discrete architecture.
3. The specific SPI command/register protocol enabling *use-without-read* of large lattice keys via block transfers.

> Each item above must be checked against prior art (§9) before any novelty is claimed. Listing a feature here is not an assertion that it is new.

## 8. Advantages over existing approaches

Low cost; keys survive host firmware compromise; quantum-safe key handling; verifiable, rapid key destruction.

## 9. Known prior art (candid assessment)

| Prior art | Overlaps with |
|-----------|---------------|
| Commercial HSMs / FIPS 140-2 & 140-3 modules | Key isolation; mandatory zeroization |
| TPM 2.0, secure elements (e.g., ATECC-class) | Hardware key isolation; use-without-read |
| NIST FIPS 203 (ML-KEM) | The cryptography itself (standardized) |
| Academic FPGA "key vault" / root-of-trust papers | FPGA-resident protected key storage |
| PKCS#11 | The use-without-read key-object model |

**Assessment:** the core concepts are well-established. OBSIDIAN is most accurately described as a novel *integration and demonstration* of known techniques, not a patentable invention. Any genuine novelty, if it exists, would lie in a narrow implementation detail and should not be assumed. This honest conclusion is itself the point of the exercise — distinguishing an engineering contribution from a patentable one.

## 10. Public disclosure status

- Public GitHub repository: `<yes / planned>`
- Senior-design expo demonstration: `<planned>`

> **Important:** a public repository and a public demo are *public disclosures*. In the U.S. they start a 12-month grace period to file a patent; in most other jurisdictions they **immediately bar** patenting. If any aspect were ever pursued for protection, the filing decision would have to precede public disclosure — so for this project, choosing to publish openly is effectively a decision *not* to patent, which is a deliberate and reasonable choice for a portfolio piece.

## 11. Possible variations / alternative embodiments

ASIC in place of the FPGA; a commercial secure element in place of the FPGA; alternative KEM or signature schemes; an encrypted channel for key transfer.

## 12. Commercial / application potential

Educational HSM platform; IoT root-of-trust reference design; teaching tool for post-quantum key management.
