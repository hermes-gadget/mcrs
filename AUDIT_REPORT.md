# MCRS Code Audit Report

**Repo audited:** `neilalexander/mcrs` (forked to `hermes-gadget/mcrs`)  
**Local clone:** `/home/ben/mcrs`  
**Commit audited:** `6c32c22` (`Update readme`)  
**Date:** 2026-07-14  
**Auditor:** Hermes Agent (kimi-k2.7-code)  
**Scope:** Full static review of the Rust source; protocol unit tests run; firmware build/check **not** run due to ESP-IDF toolchain missing and resource constraints.

---

## 1. Executive Summary

`mcrs` is a standalone **MeshCore-compatible repeater firmware** written in Rust for ESP32-S3 based Heltec boards. It is architecturally simpler than the companion builds you have been working on: no LVGL UI, no companion BLE bridge, no multi-screen navigation. Instead it provides:

- A LoRa packet forwarder/repeater using the MeshCore wire protocol.
- A serial + Telnet CLI for local and remote administration.
- Wi-Fi STA with optional NTP clock sync.
- OTA update support using the ESP-IDF bootloader partition layout.

**Overall verdict:** The codebase is small, well-structured, and the protocol crate is solid (22 unit tests pass, fuzz targets present). The firmware layer is pragmatic but has several sharp edges typical of early embedded Rust code: a number of `unwrap`/panic paths, some silent error swallowing in platform code, and one notable gap — **battery monitoring is stubbed out on Heltec V3/V4**. No critical security flaws were visible, but a few areas warrant hardening before field deployment.

---

## 2. Project Structure

| Component | Language | Lines (approx) | Purpose |
|-----------|----------|----------------|---------|
| `protocol/` | Rust no-std+alloc | ~3,500 | MeshCore packet wire format, crypto primitives, routing helpers |
| `firmware/` | Rust no-std+alloc, ESP32-S3 | ~9,000 | Main app, CLI, radio driver, board support, OTA, storage |
| `fuzz/` | Rust | ~120 | `libfuzzer` round-trip targets for protocol crate |
| `.github/workflows/` | YAML | 3 files | CI: unit tests, Docker image build, minimal rust build |

### 2.1 Key files read
- `protocol/src/lib.rs`, `packet.rs`, `header.rs`, `route_path.rs`, `crypto.rs`, `tests.rs`, plus all payload/plaintext modules.
- `firmware/src/main.rs`, `app/mod.rs`, `app/cli.rs`, `app/config.rs`, `app/regions.rs`, `app/neighbours.rs`, `app/discovery.rs`, `app/ota.rs`, `app/crypto.rs`, `app/remote.rs`, `app/telnet.rs`, `app/periodic.rs`, `app/protocol_log.rs`.
- `firmware/src/platform/esp.rs`, `platform/storage.rs`, `memory.rs`.
- `firmware/src/board/heltec.rs`, `board/heltec_v3.rs`, `board/heltec_v4.rs`, `board/mod.rs`.
- `firmware/src/modules/sx1262.rs`, `modules/spi_device.rs`, `modules/ssd1306.rs`, `modules/mod.rs`.
- `firmware/Cargo.toml`, `protocol/Cargo.toml`, `build.rs`, `Makefile`, `espflash.toml`, partition CSVs.

---

## 3. Comparison with MeshCore / Companion Builds

### 3.1 What is shared
- **Wire protocol:** `mcrs` reuses the MeshCore binary packet layout (same header, path hashes, route types, payload kinds, transport codes). It imports `mcrs-protocol` as a local crate; this is the same conceptual protocol used in companion firmware.
- **Crypto primitives:** Ed25519 identities, X25519 ECDH, ChaCha20Poly1305 (firmware) / AES-CTR (protocol advert signature verification), HMAC-SHA256. The key derivation paths look MeshCore-consistent.
- **Routing concepts:** `FLOOD`, `TRANSPORT_FLOOD`, `DIRECT`, `TRANSPORT_DIRECT`, normal vs. trace paths, deduplication via seen-packet cache, TTL/hop limits.
- **Region/scope model:** The firmware has an elaborate region allow/deny/capture list (`app/regions.rs`) mirroring MeshCore’s regional LoRa configuration.

### 3.2 What is different from companion builds
| Aspect | Companion builds (your recent work) | `mcrs` repeater |
|--------|-------------------------------------|-----------------|
| **Target hardware** | T-Deck (ESP32-S3, LovyanGFX, T-Deck keyboard, BLE companion) | Heltec V3/V4/WSL3 (SX1262 radio, OLED, push buttons) |
| **UI** | LVGL multi-screen GUI | 128×64 SSD1306 status page + serial/Telnet CLI |
| **Runtime** | SigurdOS loop + LVGL + async tasks | `embassy` executor on ESP32-S3 |
| **Connectivity** | BLE companion phone app, sometimes Wi-Fi | Wi-Fi STA only (for NTP + Telnet), serial CLI |
| **Role** | End-user chat node / companion gateway | Repeater / infrastructure node |
| **OTA** | Not present in recent companion work | Built-in OTA via ESP-IDF bootloader partitions |
| **Memory model** | PSRAM + internal RAM, LVGL heap concerns | Custom linked-list-first heap, explicit Wi-Fi allocator hooks |

### 3.3 Protocol fork status
`mcrs` does **not** use the `meshcore-dev/MeshCore` repository as a submodule. It has its own `mcrs-protocol` crate that implements the MeshCore binary format independently. This means:
- Bug fixes or protocol changes in upstream MeshCore will not flow in automatically.
- Conversely, it is self-contained and avoids submodule versioning headaches.
- The protocol crate is clean enough that keeping it in sync with upstream should be mechanical.

---

## 4. Protocol Crate Audit

### 4.1 Strengths
- **No unsafe code.**
- **No `std` dependency** (uses `alloc`).
- **Comprehensive unit tests:** 22 tests covering round-trips, truncation rejection, advert signature verification, path forwarding mutations, seen-packet cache TTL, malformed packet handling. **All pass.**
- **Fuzz targets** for both full packet decoding and individual payload decoders.
- **Defensive parsing:** most decoders return `Error::Truncated` or `Error::InvalidLength` rather than panicking.
- **Consistent error handling:** `Result<T>` alias to `Error`.
- **Round-trip invariants:** fuzz target asserts decode→encode→decode equality.

### 4.2 Findings / Concerns
1. **`Packet::decode` with raw custom payloads**  
   `PayloadKind::RawCustom` consumes all remaining bytes without length prefixing. This is fine for the reserved-nibble slot but means a `RawCustom` packet cannot be reliably re-encoded if the decoder does not know the intended length. Acceptable because it is explicitly a custom escape hatch.

2. **Protocol-level `unwrap_or` in `TextLine::as_str`**  
   Only one `unwrap_or` in the protocol crate, in SSD1306 display text rendering (`protocol` crate does not contain it; it is in firmware display driver). Protocol crate itself is essentially panic-free.

3. **`AdvertPayload::verify_signature` uses the public key embedded in the advert**  
   This is by design for MeshCore-style self-signed adverts, but it means an advert can only prove consistency with its own embedded key, not binding to a trusted root. Documented and acceptable for a decentralized mesh.

4. **`trace_hash_size` bits**  
   The 2-bit encoding reserves value `3` as invalid. Fine, but worth noting if future protocol versions want a larger hash.

---

## 5. Firmware Crate Audit

### 5.1 Architecture
The firmware is an `embassy` async application with roughly these tasks:
- Radio TX/RX loop (`board/heltec.rs` + `app/mod.rs`).
- Serial CLI input (`platform/esp.rs` UART interrupt + `app/cli.rs`).
- Periodic 4-hour zero-hop advert (`app/periodic.rs`).
- Optional Wi-Fi STA + NTP + Telnet CLI (`board/heltec.rs` + `app/telnet.rs`).
- OTA background task (`app/ota.rs`).

State is held in `AppContext`, which wraps configuration, identity, neighbour table, and outbound queue behind async-capable accessors.

### 5.2 Strengths
- **Clean separation:** protocol / platform / board / app modules are well bounded.
- **Configuration persistence:** text-based `app.conf` stored in NVS, with atomic writes and redaction of secrets for remote display.
- **Region model:** flexible allow/deny/default region handling for cross-border repeaters.
- **Duty-cycle enforcement:** firmware tracks TX airtime and enforces a configurable duty-cycle percentage.
- **OTA support:** uses `esp-bootloader-esp-idf` to select the next boot app from a firmware image.
- **RTC retained wall-clock:** uses ESP32-S3 LPWR retention registers to keep time across soft reboots.

### 5.3 Findings

#### F1 — `unwrap`/`expect`/`unreachable!` in firmware (low-to-medium)
Only a handful exist, but they are all in firmware code:

| File | Line | What | Risk |
|------|------|------|------|
| `app/config.rs` | 1234, 1244, 1254, 1263, 1271, 1276 | `.expect(...)` in unit tests only | None (tests) |
| `board/heltec.rs` | 426, 434 | `unreachable!()` after exhaustive radio kind match | Panic if new variant added and not handled |
| `app/cli.rs` | 1399 | `.unwrap_or(0)` in `format_scaled` | Harmless display formatting |
| `app/config.rs` | 507 | `fit_node_name(...).unwrap_or_else(|| String::from("Repeater"))` | Harmless fallback |

**Recommendation:** Replace the `unreachable!()` calls in `board/heltec.rs` with a safe default or error return. They are only safe as long as the `RadioKind` enum stays unchanged.

#### F2 — Silent error swallowing in platform/display code (medium)
`firmware/src/modules/ssd1306.rs` defines:
```rust
fn pin_error<E>(_error: E) {}
fn i2c_error<E>(_error: E) {}
fn unit_error<E>(_error: E) {}
```
All display and power-pin errors are silently dropped. On a repeater the OLED is non-critical, but if the reset/power GPIO fails the display will simply stay blank with no diagnostic.

**Recommendation:** At minimum log display init/write failures once per boot, or bubble errors to `app/mod.rs` so the status LED/boot message can indicate a fault.

#### F3 — Battery monitoring is stubbed (medium)
`platform/esp.rs`:
```rust
pub fn battery_level_percent() -> Option<u8> { None }
pub fn battery_millivolts() -> Option<u16> { None }
```
The status display and CLI always report “Battery not known.” Heltec V3/V4 boards have ADC-capable battery sense pins; this is a missing feature for an outdoor repeater.

**Recommendation:** Implement ADC battery sensing in `board/heltec_v3.rs`/`heltec_v4.rs` and wire it into the platform abstraction.

#### F4 — Telnet CLI enabled by plaintext password in config (medium)
Remote Telnet is disabled by default (`wifi.telnet = false`) and protected by `remote_cli_password`, but the password is stored in plaintext in NVS and transmitted in plaintext over TCP. There is no TLS or challenge-response for the TCP session.

**Recommendation:** This is acceptable for a local maintenance LAN but should be documented as insecure for untrusted networks. Consider restricting Telnet to the local subnet or requiring a one-time setup step.

#### F5 — Remote CLI authentication is password-only over an encrypted MeshCore side-channel (acceptable)
The mesh-based remote CLI (`app/remote.rs`, `app/cli.rs`) uses ECDH + ChaCha20Poly1305 for request/response encryption, which is a significant step up from Telnet. The password is compared in plaintext inside the decrypted payload. This matches MeshCore companion behavior and is acceptable for the threat model.

#### F6 — `unsafe` usage is localized and documented (low)
All 13 `unsafe` blocks are in `platform/esp.rs`: peripheral stealing, heap initialization, Wi-Fi allocator hooks, RTC register writes, panic reboot export. Each has a safety comment. No `unsafe` elsewhere.

#### F7 — Cargo.toml feature gating mismatch for host tests (low)
The firmware binary tests cannot compile on the host because ESP-only dependencies (`embassy-net`, `esp-bootloader-esp-idf`) are used unconditionally inside board code when no `esp32`/`esp32s3` feature is enabled. This is expected for an ESP target, but it means CI unit tests cannot exercise firmware logic without a mock platform.

**Recommendation:** Move host-testable logic (config parsing, region tables, CLI parser helpers) into a `#[cfg(test)]` module or a small `firmware-util` library that compiles on host. Currently only `app/config.rs` has tests, but they do not run on CI because the binary fails to compile for host.

#### F8 — Partition tables are conservative (informational)
`firmware/partitions_heltec_v3.csv` and `partitions_heltec_v4.csv` split 4 MB flash into roughly 1 MB factory + 1 MB OTA + NVS + etc. Reasonable for Heltec boards. If you later target 8 MB modules, revisit.

#### F9 — CI is minimal (informational)
`.github/workflows/rust.yml` is only 11 lines. It presumably runs `cargo test` but will fail on firmware because of the ESP toolchain. The unit-tests workflow only runs `--workspace` on push; on GitHub’s default runner this will likely fail for the firmware package.

**Recommendation:** Pin CI to run only `cargo test -p mcrs-protocol` and add a separate workflow that uses `esp-rs/xtensa-toolchain` for firmware checks. Do not run full firmware builds on every push in shared CI unless you have large runners.

---

## 6. Security Notes

| Topic | Assessment |
|-------|------------|
| **Identity/key storage** | Private key seed stored in NVS as raw bytes. No passphrase encryption. Standard for embedded MeshCore nodes, but be aware that physical NVS dump exposes identity. |
| **Remote CLI encryption** | ECDH + ChaCha20Poly1305 over mesh direct messages. Good. |
| **Telnet** | Plaintext password + plaintext TCP. Avoid on untrusted networks. |
| **OTA** | Receives firmware image via mesh multipart packets and writes to ESP-IDF OTA partition. OTA packets are encrypted/authenticated through the normal mesh path. No rollback pin or signature verification visible beyond MeshCore-level auth. Acceptable for a hobby repeater; for production add a code-signing check. |
| **Replay protection** | Timestamps used in request plaintext; remote CLI rejects old timestamps (`app/remote.rs`). Reasonable. |
| **Flood mitigation** | Hop limits, TTL, seen-packet cache, duty cycle, region allow/deny. Good surface area. |

---

## 7. Build / Test Results

| Check | Result |
|-------|--------|
| `cargo test -p mcrs-protocol` | ✅ 22 passed, 0 failed |
| `cargo test -p mcrs-firmware` | ❌ Fails on host (ESP-only deps, no `esp32s3` feature selected) |
| `cargo +esp check-heltec-v3` | ❌ Toolchain `esp` not installed on this host |
| `cargo audit` | ⚠️ Could not verify yanked status because several ESP/embassy crates are not in local crates.io index; no actual vulnerabilities reported |
| `cargo clippy` | Not run (requires successful check) |
| `cargo fmt` check | Not run (formatting only) |

---

## 8. Recommendations (Prioritized)

1. **Fix board `unreachable!()` paths** to avoid future radio-kind additions causing panics.
2. **Log display errors** instead of silently dropping them.
3. **Implement battery ADC sensing** for Heltec V3/V4.
4. **Document Telnet security model** or restrict it to trusted networks.
5. **Split host-testable firmware logic** into a library so CI can run config/region/parser tests.
6. **Update CI** to use ESP toolchain only for firmware, and run protocol tests on every push.
7. **Add OTA image signature verification** if this repeater is intended for unattended deployment.
8. **Pin `esp-hal` version carefully** — currently on `=1.0.0-rc.0`, an RC. Track final 1.0 release.

---

## 9. Bottom Line

`mcrs` is a credible, compact MeshCore repeater implementation. The protocol crate is production-quality for its scope, and the firmware is a sensible first-generation embedded Rust app. It is **not** at the same maturity level as the companion/LVGL builds you have been testing — it lacks on-device validation polish (battery, display error handling, host-testable firmware logic) — but the core routing/repeating logic is sound and the codebase is easy to reason about.

If you plan to deploy it, I would prioritize the battery sense and CI/test split before adding new features.
