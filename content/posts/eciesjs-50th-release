+++
title = "eciesjs 0.5.0: Seven Years, Fifty Releases"
date = "2026-03-12"
cover = ""
tags = ["JavaScript", "cryptography"]
description = "The 50th release of the quietly influential JavaScript ECIES library."
showFullContent = false
+++

*The 50th release of the quietly influential JavaScript ECIES library.*

## The Journey

On November 27, 2018, I published `eciesjs` v0.0.1 to npm. It was a small, focused library — a JavaScript counterpart to [eciespy](https://github.com/ecies/py) — born from a simple gap: the JavaScript ecosystem needed a clean, correct Elliptic Curve Integrated Encryption Scheme implementation.

Seven years and fifty releases later, the core API is still two functions: `encrypt` and `decrypt`. That simplicity is deliberate. Cryptographic libraries should be boring — do one thing correctly and get out of your way.

What has changed is everything underneath. The library evolved from secp256k1 and aes-256-gcm to supporting curve25519 (x25519 and ed25519) and xchacha20-poly1305. It went from Node.js-only to running on every major JavaScript runtime — Node.js, Bun, Deno, browsers, and React Native. Dependencies were pruned, re-evaluated, and replaced with audited alternatives. Through all of it, the surface area stayed small: about 400 lines of source code.

Version 0.5.0 marks the 50th published release. No confetti — just steady, principled work.

## What's New in 0.5.0

This is a **breaking change** release with a clear rationale.

**`encrypt` and `decrypt` now always return `Uint8Array`.**

Why? Because `Uint8Array` is the universal byte type in modern JavaScript. Node.js, Deno, Bun, and browsers all support it natively. As the ecosystem moves away from Node-specific types, eciesjs follows.

**Migration is trivial.** If your code relied on `Buffer`-specific methods:

```ts
// Before (v0.4.x on Node.js/Bun/Deno): result was Buffer
const result = decrypt(sk, data);
result.toString("hex"); // Buffer method

// After (v0.5.0): result is always Uint8Array
const result = decrypt(sk, data);
Buffer.from(result).toString("hex"); // wrap if you need Buffer
```

Other changes in this release:

- The deprecated `utils` re-export from the main entry point has been removed. Use `import { ... } from "eciesjs/utils"` instead
- Other deprecated APIs are removed as well
- Dependency updates across the board

## Adoption

eciesjs now sees roughly **12.5 million monthly downloads** on npm. What began as a niche cryptographic utility has, quietly, become infrastructure.

Notable adopters include:

- **[MetaMask SDK](https://github.com/MetaMask)** — the mobile crypto wallet that millions of users interact with daily
- **[dotenvx](https://dotenvx.com)** — a next-generation dotenv tool and an eciesjs sponsor

Beyond these, dozens of libraries and applications across the npm ecosystem depend on eciesjs for public-key encryption. You can browse the full list of [dependents on npm](https://www.npmjs.com/browse/depended/eciesjs).

The library also belongs to a broader cross-language family: [Python](https://github.com/ecies/py), [Rust](https://github.com/ecies/rs), [Go](https://github.com/ecies/go) and compatible implementations in other languages — all sharing the same cryptographic protocol.

## Security and the Case for an Audit

Security has always been a first principle. eciesjs has only three runtime dependencies, all from Paul Miller's audited **noble** family:

- [noble-curves](https://github.com/paulmillr/noble-curves) — elliptic curve operations
- [noble-hashes](https://github.com/paulmillr/noble-hashes) — cryptographic hash functions
- [noble-ciphers](https://github.com/paulmillr/noble-ciphers) — symmetric ciphers

These are complemented by `node:crypto` for native performance where available. No native bindings, no hidden complexity.

Every npm release is built on GitHub Actions with verifiable [provenance attestations](https://www.npmjs.com/package/eciesjs#provenance). The codebase is small enough to audit in an afternoon.

**But "auditable" is not the same as "audited."**

With 12.5 million monthly downloads and adoption by projects like MetaMask, a professional third-party security audit would benefit the entire dependency chain. I am actively seeking sponsorship to make this happen.

If your product or organization depends on eciesjs — or if you simply value secure open-source cryptography — please consider [sponsoring this effort](https://github.com/sponsors/kigawas). An audit protects not just eciesjs, but every application built on top of it.

## Looking Ahead

The direction hasn't changed: keep eciesjs small, correct, and reliable. There are no plans to bloat the API or chase features for their own sake.

I want to thank the community for seven years of trust, [dotenvx](https://dotenvx.com) for their sponsorship, [Savely Krasovsky](https://github.com/savely-krasovsky) for reaching out and implementing the Go version and [Paul Miller](https://github.com/paulmillr) for the noble-\* libraries that form eciesjs's cryptographic foundation.

Here's to the next fifty.
