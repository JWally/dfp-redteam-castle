# Castle on x.com — production deployment: live tamper + token cipher reversed

Follow-up on the vanilla-SDK assessment (`CASTLE_ASSESSMENT_2026_10_26.md`),
covering the **production Castle deployment on x.com** (chunk
`ondemand.castle.c42626ea.js`, captured 2026-05-21/22). Nearly all of this
moves **read** and **tamper** to 🟢 on the live deployment — not just the
vanilla SDK. Full archive writeup: `artifacts/X.Castle.md`.

## 1. What the production build changed (vs vanilla SDK)

- The static structural regex (`wS[Hg]` 4-arg encoder body fingerprint) from
  the baseline analysis is **dead** against the new build — 0 patched chunks
  across `/i/flow/login`, `/login`, `/signup`, `/`.
- Per-field **encoder distribution**: the runtime token now uses *distributed
  byte-emitter helpers* — `f4`, `tN`, `lb`, `lM` — each applying its own
  bit-mix. **No plaintext at any `btoa` boundary.**
- Native-API alias table + arithmetic-obfuscated string decoders (`tn()` /
  `r9()`) resolve identifiers at load time.

## 2. New attack: input-side value tampering (LIVE, x.com)

The bundle still has to read signals from **unprivileged JS-engine surfaces**.
Patch on the **prototype** (both access paths resolve to the same descriptor,
so Castle's cross-validation is satisfied automatically), and
`Object.defineProperty` lies land verbatim:

| Signal | Baseline (real) | Tampered (what Castle read) | Landed? |
|---|---|---|---|
| `navigator.webdriver` | `true` (headless leak) | `undefined` | **yes** |
| `navigator.platform` | `Linux x86_64` | `Win32` | yes |
| `navigator.hardwareConcurrency` | `12` | `8` | yes |
| `navigator.deviceMemory` | `8` | `16` | yes |
| `navigator.languages` | `["en-US"]` | `["en-US","en"]` | yes |
| `navigator.plugins` | `{}` | 5 PDF Viewer entries | yes |
| `navigator.mimeTypes` | `{}` | 2 PDF mimes | yes |
| `navigator.userAgentData` | `{}` | Chrome-Win brands + versions | yes |
| `screen.width/height` | 1280×800 | **1920×1080** | yes |
| `screen.availWidth/Height` | 1280×800 | 1920×1080 | yes |
| `WebGL.getParameter(37446)` | `SwiftShader Device (Subzero)…` | `ANGLE (NVIDIA, RTX 3060 …)` | **yes** |

**13 lies landed verbatim**, including the two most actionable bot tells:
`webdriver true→undefined` and `SwiftShader → NVIDIA RTX 3060` (reclassifies
"datacenter/VM" → "consumer desktop").

### Token still builds and submits

| | Baseline | Tampered |
|---|---|---|
| Castle btoa calls / session | 1,345 | 1,345 |
| Largest btoa input (binary) | 10,953 B | 10,772 B |
| Final token in `POST /1.1/onboarding/task.json` body | 14,544 base64 chars | **14,308 chars** |
| Flow-init token | 352 chars | 352 chars |
| `createRequestToken` errored? | no | no |
| Submit reached x.com? | **yes** | **yes** |

The tampered token is ~236 chars smaller (spoofed values serialize tighter) —
the lies are *in* the token, and x.com forwards it server-to-server to
Castle's `/v1/filter`.

## 3. Token cipher REVERSED — Castle ships its own decryptor

Without reversing anything algebraically: **Castle's own inverse function sits
in the same bundle** (used for build-time string deobfuscation) and decrypts
the runtime token **byte-for-byte**.

`tt()` encrypt / `tv()` decrypt — 16-bit-lane bit-mix, 7 stages:

```
tt:  n = (n-54655); n ^= 46488; n = (n*54213+385); n = rotR12(n); n += 60834; n = rotR11(n)
tv:  n = rotL11(n); n -= 60834; n = rotL12(n); n = (n-385)*13069; n ^= 46488; n += 54655
     (54213 * 13069 ≡ 1 mod 2^16)
```

*Each stage of `tv()` is the arithmetic inverse of `tt()`, applied in reverse
order.* Full implementation + self-tests: `scripts/castle-cipher.mjs`.

### What a decrypted token contains (TLV)

The 1,208-char plaintext out of `tv(token)` is a **TLV header**
(`(sub_tag << 3) | type`, the May-15 wire format) + **~25 concatenated
per-field base64 chunks**. `scripts/castle-tt-instrument.mjs` captures the
per-field plaintexts directly at the `tt()` boundary — e.g. the token-gen
time `"1779449228994"` (epoch ms), 8-char hex signal hashes (`902910fc`,
`f394e296`…), empty-collection literals, and the aggregate payload.

### What this unlocks

| Capability | Status |
|---|---|
| Decrypt any captured Castle token → plaintext TLV | **YES** (`scripts/decrypt.mjs`) |
| Read per-field plaintexts (hashes, timestamp, literals) | **YES** (bundle patch) |
| **Forge tokens by running `tt()` forward on chosen inputs** | **YES** — `scripts/castle-cipher.mjs` exports `tt(input)` |
| Decode the *inner* per-field cipher (embedded base64s in the aggregate) | NO — a different bit-mix (`f4/tN/lb/lM`); ~1–2 days similar RE |
| Risk-score visibility (`/v1/filter` response) | NO — needs merchant secret key |

## 4. Honest boundaries (still NOT beaten)

- **Inner per-field cipher not reversed** — the embedded base64 chunks in the
  aggregate use a different bit-mix than `tt()`; their inverses likely also
  sit in the bundle as string-deobfuscators (~1–2 days to extract).
- **No undetectable token forgery**: Castle correlates session-activity
  accumulation across fires, and the customer-backend relay gives Castle
  server-side IP/TLS/header visibility the client can't influence. A forged
  token that doesn't match the live HTTP context fails.
- **The content we show is still honestly a bot** unless canvas/
  `performance.now()`/`__cuid` are also spoofed; and
- **No `mousemove`/keyboard shaping** — the session-activity accumulator
  channel is untouched.

## 5. Assessment (revised, production)

- **Read payload:** 🟢 — token **fully decryptable** client-side (bundle's own
  `tv()`); per-field plaintexts captured; TLV structure mapped.
- **Tamper with payload:** 🟢 — **13 prototype-level lies landed verbatim in a
  cryptographically valid, submitted token** on the live x.com deployment.
- **Mint token:** 🟢/🟡 — token **forgeable** by running `tt()` forward
  (`scripts/castle-cipher.mjs`); caveat: Castle's server-side session
  accumulation + relay-side IP/TLS context still gate the endpoint.
- Ratio unchanged intellectually: the token proves "the SDK ran on inputs it
  read", and both inputs **and** the encoder are now attacker-controlled.
