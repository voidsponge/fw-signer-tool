# fw-signer-tool

> **Firmware Signer (White Zone)** — an offline desktop & CLI tool to sign firmware updates (`.bin`) with strong cryptography, suitable for air-gapped environments and factory floors. ([GitHub][1])

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](#license)

*Status: pre-release / WIP*

---

## Why

Firmware updates are a prime supply-chain target. This tool produces a **signed, canonical manifest** and detached signature for your `.bin` image, enabling devices to **verify authenticity** and **enforce anti-rollback** before applying an update — even when the update is delivered via **USB** in a **white/air-gapped zone**.

---

## Key Features

* **Air-gapped by design**: no network access required for signing.
* **Strong crypto**:

  * **Ed25519** signatures (with **Argon2id** passphrase derivation), or
  * **YubiKey PIV (ECDSA P-256)** hardware-backed signatures.
* **Canonical JSON manifest**:

  * Global SHA-256 of the `.bin`
  * **Chunk hashes** (resume/re-verify on target)
  * `product`, `component`, `version`, `min_hw_rev`, timestamp
* **Safe update flows**: ready for **A/B slots** and **anti-rollback** on device.
* **Desktop GUI** (Rust + egui) **and** CLI variants.
* **White-zone friendly UX**: select file → sign → export `manifest.json` + `manifest.sig`.

> The repository currently contains the project scaffolding and MIT license. This README documents the intended capabilities and usage. ([GitHub][1])

---

## Security Model (high-level)

* **Trust anchor lives on the device** (ROM/OTP/secure element). The device verifies signatures **locally**; this tool only signs.
* **Manifest is canonical** (no pretty-print) to ensure signature stability.
* **Anti-rollback**: the device compares `version` to a monotonic counter (OTP/TPM NV/SE).
* **Defense in depth** on target: verify signature → verify global hash → verify chunk hashes → write to inactive slot → self-test → switch.

> A TPM **does not** enforce signature policy by itself; your bootloader/firmware must implement verification and rollback policy.

---

## Outputs

After signing `BOARD123_bios_v0017.bin`, you’ll get:

```
BOARD123_bios_v0017.manifest.json   # canonical manifest (binary JSON, stable)
BOARD123_bios_v0017.manifest.sig    # detached signature (hex for Ed25519, DER hex for ECDSA)
```

**Manifest fields (example):**

```json
{
  "product": "BOARD123",
  "component": "bios",
  "version": 17,
  "min_hw_rev": 2,
  "created_at": "2025-10-12T10:31:00Z",
  "sign_alg": "Ed25519",
  "image": {
    "path": "BOARD123_bios_v0017.bin",
    "size": 2097152,
    "sha256": "..."
  },
  "chunks": [
    { "off": 0, "len": 65536, "sha256": "..." }
  ]
}
```

---

## Build (Desktop App)

Requirements:

* Rust toolchain (`rustup`)
* For YubiKey signing: PC/SC runtime (`pcscd` on Linux, YubiKey Manager on Windows/macOS)

```bash
git clone https://github.com/voidsponge/fw-signer-tool
cd fw-signer-tool
cargo build --release
./target/release/fw-signer-gui
```

**Platforms:** Windows / macOS / Linux (egui/eframe).
(You can later package as `.exe`, `.app`, or `.AppImage`.)

---

## Build (CLI)

If you prefer minimal tooling:

```bash
cargo build --release
./target/release/fw-signer --help
```

Typical usage:

```bash
# 1) Generate keys (Ed25519)
./fw-signer keygen \
  --out-seed keys/ed25519.seed \
  --out-pub  keys/ed25519.pub

# 2) Sign a .bin
./fw-signer sign \
  --bin dist/BOARD123_bios_v0017.bin \
  --seed keys/ed25519.seed \
  --out-dir dist \
  --product BOARD123 \
  --component bios \
  --version 17 \
  --min-hw-rev 2 \
  --chunk-size 65536
```

Outputs: `…manifest.json` and `…manifest.sig`.

---

## Using a YubiKey (PIV)

* Insert YubiKey with a **PIV key/cert** in **slot 9C** (ECDSA P-256).
* In the GUI, select **YubiKey PIV** and enter **PIN** when prompted.
* The tool signs the manifest digest on the key; your **private key never leaves** the token.
* The tool can export the PIV **certificate** (DER, base64) so your devices can pin/verify the public key.

*Tip:* You can also use 9A/9E in the future — the code can be extended to support additional slots.

---

## Air-Gapped Workflow (White Zone)

1. Move the `.bin` to the signing workstation via clean media.
2. Launch the **offline GUI** (or CLI) and sign:

   * Option A: **Passphrase → Ed25519** (derived with Argon2id)
   * Option B: **YubiKey PIV** (ECDSA P-256)
3. Deliver `…manifest.json` and `…manifest.sig` alongside the `.bin`.
4. On the device, the bootloader must:

   * Verify signature using **anchored public key**
   * Check `product`, `min_hw_rev`, **version** (anti-rollback)
   * Verify **global** + **chunk** hashes before switching slots

---

## Roadmap

* [ ] Dual signatures (e.g., Ed25519 **and** YubiKey PIV) with `signatures: []` array
* [ ] PKCS#11 / HSM support (Nitrokey HSM, SoftHSM)
* [ ] QR export of signatures (no USB)
* [ ] One-click packaging (Windows .exe, macOS .app, Linux .AppImage)
* [ ] Verification helper (desktop/CLI) for QA benches
* [ ] Example bootloader verification snippets (C / Rust `no_std`)

---

## Contributing

PRs and issues are welcome! Please:

* Keep changes small and security-focused
* Include threat considerations in PR descriptions
* Avoid adding network dependencies or telemetry

---

## License

This project is licensed under the **MIT License**. See [LICENSE](./LICENSE). ([GitHub][1])

---

## Disclaimer

This tool **signs** artifacts. Your device firmware must **enforce** signature verification, anti-rollback, and A/B switching. Always review cryptographic choices with your security team and test on **sacrificial hardware** before production rollout.

[1]: https://github.com/voidsponge/fw-signer-tool "GitHub - voidsponge/fw-signer-tool: Firmware Signer (White ZONE)"
