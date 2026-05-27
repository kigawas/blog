+++
title = "eciesjs 0.5.0: Seven Years, Fifty Releases"
date = "2026-04-03"
cover = ""
tags = ["JavaScript", "cryptography"]
description = "The 50th release of the quietly influential JavaScript ECIES library"
showFullContent = false
+++

## The Journey

On November 28, 2018, I published `eciesjs` v0.0.1 to [npm](https://www.npmjs.com/package/eciesjs/v/0.1.0). It was a small, focused library — a JavaScript/TypeScript counterpart to [eciespy](https://github.com/ecies/py) — born from a simple gap: the JavaScript ecosystem needed a clean, correct Elliptic Curve Integrated Encryption Scheme implementation.

Seven years and fifty releases later, the core API is still two functions: `encrypt` and `decrypt`. That simplicity is deliberate. Cryptographic libraries should be boring — prevent misuse and get out of your way.

What has changed is everything underneath. The library evolved from secp256k1 and aes-256-gcm to supporting curve25519 (x25519 and ed25519) and xchacha20-poly1305. It went from Node-only to running on every major JavaScript runtime — Node, Bun, Deno, browsers, and React Native. Dependencies were reduced, re-evaluated, and replaced with audited alternatives. Through all of it, the complexity is contained: there are only less than 300 lines of source code.

Version 0.5.0 marks the 50th published release. No confetti — just steady, principled work.

## What's New in 0.5.0

This is a **breaking change** release with a clear rationale.

**`encrypt` and `decrypt` now always return `Uint8Array`.**

Why? Because `Uint8Array` is the universal byte type in modern JavaScript. Node, Deno, Bun, and browsers all support it natively. As the ecosystem moves away from Node-specific types, eciesjs follows.

**Migration is trivial.** If your code relied on `Buffer`-specific methods:

```ts
// Before (v0.4.x on Node/Bun/Deno): result was Buffer
const result = decrypt(sk, data);
result.toString("hex"); // Buffer method

// After (v0.5.0): result is always Uint8Array
const result = decrypt(sk, data);
Buffer.from(result).toString("hex"); // wrap if you need Buffer
```

Other changes in this release:

- The deprecated `utils` re-export from the main entry point has been removed. Use `import { ... } from "eciesjs/utils"` instead
- Other deprecated APIs are removed as well
- Refactoring and bug fixes in `utils` folder. There are some breaking changes here, but should not affect most users
- Dependency updates across the board

## Adoption

eciesjs now sees roughly **17 million monthly downloads** on npm. What began as a niche cryptographic utility has, quietly, become infrastructure.

Notable adopters include:

- **[MetaMask SDK](https://github.com/MetaMask)** — the mobile crypto wallet that millions of users interact with daily
- **[dotenvx](https://dotenvx.com)** — a next-generation dotenv tool and an eciesjs sponsor

Beyond these, dozens of libraries and applications across the npm ecosystem depend on eciesjs for public-key encryption. You can browse the full list of [dependents on npm](https://www.npmjs.com/browse/depended/eciesjs).

The library also belongs to a broader cross-language family: [Python](https://github.com/ecies/py), [Rust](https://github.com/ecies/rs), [Go](https://github.com/ecies/go) and compatible implementations in other languages — all sharing the same cryptographic protocol.

The ecosystem has even reached [academia](https://github.com/ecies/ecies.github.io?tab=readme-ov-file#academic-papers-with-ecies-libraries-reference).

## Security and the Case for an Audit

Security has always been a first principle. eciesjs has only three runtime dependencies, all from Paul Miller's audited **noble** family:

- [noble-curves](https://github.com/paulmillr/noble-curves) — elliptic curve operations
- [noble-hashes](https://github.com/paulmillr/noble-hashes) — cryptographic hash functions
- [noble-ciphers](https://github.com/paulmillr/noble-ciphers) — symmetric ciphers

These are complemented by `node:crypto` for native performance where available through an internal adapter, [@ecies/ciphers](https://github.com/ecies/js-ciphers). No native bindings, no hidden magic.

Every npm release is built on GitHub Actions with verifiable [provenance attestations](https://www.npmjs.com/package/eciesjs#provenance). The codebase is compact enough to audit in an afternoon.

**But "auditable" is not the same as "audited."**

With millions of weekly downloads and adoption by projects like MetaMask, a professional third-party security audit would benefit the entire dependency chain. I am actively seeking sponsorship to make this happen.

If your product or organization depends on eciesjs — or if you simply value secure open-source cryptography — please consider [sponsoring this effort](https://github.com/sponsors/kigawas). An audit protects not just eciesjs, but every application built on top of it.

## Looking Ahead

The direction hasn't changed: keep eciesjs clean, correct, and reliable. There're no plans to bloat the API or chase features for their own sake.

I'm grateful to the community for years of trust, [Scott Motte](https://github.com/motdotla) and [dotenvx](https://dotenvx.com) for their sponsorship.

Thanks also to [Savely Krasovsky](https://github.com/savely-krasovsky) for reaching out and implementing the Go version, and [Paul Miller](https://github.com/paulmillr) for suggestions on switching to the noble-\* libraries that form eciesjs's cryptographic foundation.

The journey keeps unfolding. Here's to the next fifty.
