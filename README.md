# mpp-rs (Hardened Fork)

Hardened fork of [tempoxyz/mpp-rs](https://github.com/tempoxyz/mpp-rs) — the Rust SDK for the [**Machine Payments Protocol**](https://mpp.dev).

This fork applies **security fixes** and **performance optimizations** to the upstream codebase.

## Changes from Upstream

### Security Fixes

| Severity | Vulnerability | File | Description |
|----------|--------------|------|-------------|
| **CRITICAL** | Timing attack on HMAC verification | `server/mpp.rs` | Challenge ID comparison used standard `!=` operator, enabling byte-by-byte HMAC reconstruction via response-time measurement. Replaced with constant-time comparison (`constant_time_eq`). |
| **HIGH** | Integer overflow in channel balance | `session_method.rs` | `spent + amount` used unchecked addition; overflow would wrap to zero and bypass balance checks. Replaced with `checked_add()` that returns an error on overflow. |
| **HIGH** | Integer overflow in SSE stream | `server/sse.rs` | `ch.spent + tick_cost` could overflow in need-voucher event computation. Replaced with `saturating_add()`. |
| **MEDIUM** | Timing attack on body digest | `body_digest.rs` | Digest verification used `==`, leaking digest value via timing side-channel. Replaced with constant-time byte comparison. |
| **MEDIUM** | FileStore path traversal | `store.rs` | No validation that resolved file paths stay within the store directory. Added `starts_with()` boundary check after sanitization. |
| **MEDIUM** | FileStore unbounded key length | `store.rs` | No limit on key length, enabling filesystem-level DoS. Added 512-byte maximum key length. |

### Performance Optimizations

| Area | File | Description |
|------|------|-------------|
| Copy semantics | `types.rs` | Derived `Copy` on `PayloadType` and `ReceiptStatus` enums, eliminating unnecessary heap clones. |
| Header formatting | `headers.rs` | Rewrote `format_www_authenticate` to build directly into a pre-allocated `String` via `write!`, replacing Vec + multiple `format!()` + `join()`. |
| Header escaping | `headers.rs` | Rewrote `escape_quoted_value` as a single-pass iterator with integrated CRLF rejection, replacing two chained `replace()` calls (2 intermediate String allocations). |
| HMAC computation | `challenge.rs` | Replaced `.join("|")` in `compute_challenge_id` with `String::with_capacity` + `write!` for pre-allocated buffer construction. |
| Unused parameter removal | `middleware.rs` | Removed unused `_message` parameter from `error_response()` and eliminated `format!()` calls at call sites that were generating strings nobody read. |
| Redundant computation | `proxy/service.rs` | Fixed `parse_route_pattern` calling `to_uppercase()` twice; now calls once and reuses the result. |
| JSON construction | `proxy/service.rs` | Replaced manual `Map::new()` + multiple `.to_string()` on static keys in `serialize_payment` with `json!` macro. |
| Character classification | `store.rs` | Replaced `is_alphanumeric()` (full Unicode) with `is_ascii_alphanumeric()` (fast ASCII-only check) in `FileStore::key_path`. |

## Overview

[MPP](https://mpp.dev) lets any client — agents, apps, or humans — pay for any service in the same HTTP request. It standardizes [HTTP 402](https://mpp.dev/protocol/http-402) with an open [IETF specification](https://paymentauth.org), so servers can charge and clients can pay without API keys, billing accounts, or checkout flows.

## Quick Start

### Server

```rust
use mpp::server::{Mpp, tempo, TempoConfig};

let mpp = Mpp::create(tempo(TempoConfig {
    recipient: "0x742d35Cc6634C0532925a3b844Bc9e7595f1B0F2",
}))?;

let challenge = mpp.charge("1")?;
let receipt = mpp.verify_credential(&credential).await?;
```

### Client

```rust
use mpp::client::{PaymentMiddleware, TempoProvider};
use reqwest_middleware::ClientBuilder;

let provider = TempoProvider::new(signer, "https://rpc.moderato.tempo.xyz")?;
let client = ClientBuilder::new(reqwest::Client::new())
    .with(PaymentMiddleware::new(provider))
    .build();

// Requests now handle 402 automatically
let resp = client.get("https://mpp.dev/api/ping/paid").send().await?;
```

## Feature Flags

| Feature | Description |
|---------|-------------|
| `client` | Client-side payment providers (`PaymentProvider` trait, `Fetch` extension) |
| `server` | Server-side payment verification (`ChargeMethod` trait) |
| `tempo` | [Tempo](https://tempo.xyz) blockchain support (includes `evm`) |
| `evm` | Shared EVM utilities (Address, U256, parsing) |
| `middleware` | reqwest-middleware support with `PaymentMiddleware` (implies `client`) |
| `tower` | Tower middleware for server-side integration |
| `axum` | Axum extractor support for server-side convenience |
| `utils` | Hex/random utilities for development and testing |

## Building & Testing

```bash
git clone https://github.com/nktkt/mpp-rs
cd mpp-rs
cargo test
```

## Upstream

- Upstream repository: [tempoxyz/mpp-rs](https://github.com/tempoxyz/mpp-rs)
- Protocol documentation: [mpp.dev](https://mpp.dev)
- IETF specification: [paymentauth.org](https://paymentauth.org)

## License

Licensed under either of [Apache License, Version 2.0](./LICENSE-APACHE) or [MIT License](./LICENSE-MIT) at your option.
