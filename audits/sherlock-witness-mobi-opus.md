# Sherlock Report: witness-mobi

**Target**: https://github.com/pwhoppa/witness-mobi
**Type**: GitHub repo (single-page PWA + marketing landing page)
**Mode**: Deep
**Investigated**: 2026-05-18

## Identification

Tiny repo. Five tracked artifacts at root: a one-line `README.md`, a dual-license `LICENSE.md` (AGPLv3 + commercial, © 2026 Individuation LLC, with the AGPLv3 body literally stubbed as `[Full standard AGPLv3 text would be here. Network fetch failed.]`), a `vercel.json` that forces `Content-Type: text/html` for `/app`, a 922-line marketing landing page `index.html`, and the actual PWA at `app/index.htmk` (812 lines, ~38 KB, single self-contained HTML file with one inline `<script>`). A loose root file `Acoustic + WebRTC Orchestrat…` contains an unreferenced JS class `CigawAcousticWebRTC` (signaling via `wss://relay.indivisa.net`) that is *not* loaded by either page. Eight commits total, all by one author, dated 2026-05-13 through 2026-05-17 (future-dated relative to most readers — author's clock, or this is freshly minted). Zero dependencies, zero build system, no tests, no CI.

## Function

Claimed: WITNESS is a browser-native PWA that captures sensor data, hashes it with SHA-256 + SHA3-256 + BLAKE2b-256, signs the composite with a hardware-bound non-extractable ECDSA P-256 key, appends each receipt to a local IndexedDB ledger chain-linked by `sha256(prev_hash ‖ body)`, and emits a QR code that anyone can verify offline against the original file. The landing page is explicit that this is a *reference implementation, not a product*, and that v0.1 is single-device only — Bluetooth peer co-signing (CIGaW), OpenTimestamps anchoring, and burn-after-use ephemeral keys are flagged as v0.2.

Actual: the PWA at `app/index.htmk` delivers v0.1 as advertised. Hand-rolled BLAKE2b (`compress`, ADD/XOR/ROT/G mixers) and Keccak (`keccakf`) sit next to `crypto.subtle.digest('SHA-256', …)`. The ECDSA keypair is generated with `extractable: false` and stored in an IndexedDB keystore — that's the closest the WebCrypto API gets to "hardware-bound," and the marketing copy's "Secure Enclave / StrongBox" framing is aspirational on the browser side (the OS *may* back it there, but the JS can't enforce it). The chain-link is real: `ledgerAll()` walks every prior entry, recomputes `sha256(prev ‖ body)`, and returns `{ok:false, brokenAt:seq}` on the first mismatch. QR generation is also from scratch — Reed-Solomon (`rsGen`, `rsEncode`), version selection, finder placement, format bits — no `qrcode.js` import. `getUserMedia` is wired for capture. `quiet` is *mentioned* in a string but no acoustic transport is actually implemented in the PWA.

Divergences worth naming: (1) the marketing page promises Bluetooth co-signing as the v0.2 mechanism, but the lone orchestrator stub at the repo root implements WebRTC over a signaling relay (`wss://relay.indivisa.net`) plus an *acoustic* discovery channel — two different proximity-attestation designs, both gestured at, neither shipped. (2) The acoustic transmit/receive methods in that stub are explicit `// Stub` comments. (3) The app file is named `index.htmk` (typo); `vercel.json` papers over it by pinning the response Content-Type. (4) `LICENSE.md` cites AGPLv3 but never actually includes the AGPLv3 text — the bracketed `[Full standard AGPLv3 text would be here. Network fetch failed.]` is sitting in production. That is not how dual-licensing works.

## Opinion

The shipped PWA is the good part of this repo, and it is genuinely good. Writing BLAKE2b, Keccak-f, and Reed-Solomon-with-finder-pattern QR in one self-contained ~38 KB HTML file with one script tag and zero npm imports is the kind of thing that signals the author can actually code, has done their homework, and meant the "zero dependencies, zero cloud calls" boast. The hash-chained IndexedDB ledger with a real verifier is more than most "tamper-evident receipt" demos ship — many would have stopped at "sign the hash" and called it a day. The honest-scope section of the landing page ("does not prove what was captured is true, complete, or contextually honest") is well-written and unusually forthright for this category.

The rest of the repo is rough. The README is a single sentence. The LICENSE file has an unfilled placeholder where the AGPLv3 body should be, which is both a legal mess and a self-inflicted credibility wound for a project whose entire pitch is integrity. The `CigawAcousticWebRTC` file at the root is design-doc-shaped — orphaned, unreferenced, conflicts with the landing page's "Bluetooth co-signing" framing, and contains explicit stubs in load-bearing methods (`transmitAcousticToken`, `receiveAcousticToken`). No tests anywhere. No verifier tool — the landing page claims "anyone with the QR and the original file can verify the receipt without contacting any server" but there is no separate verifier shipped; the user has to trust that another instance of this PWA, or someone's home-grown script, will reconstitute the same composite hash and signature check. For a project whose value proposition is "you don't have to trust us," that's a meaningful gap.

Who it is for: someone evaluating browser-native tamper-evidence techniques, writing about content provenance, or looking for a clean reference implementation of WebCrypto + IndexedDB + chain-linked ledger + scratch-built QR in one file. Read `app/index.htmk` and you'll learn something. Who it is not for: anyone who wants to *use* this for consequential evidence today. The author says this on the tin and they mean it. Alternatives in the same space worth knowing: Adobe/Microsoft's C2PA / Content Authenticity Initiative (industry-backed, ecosystem-supported, much heavier), ProofMode by Guardian Project (Android, mature, actively maintained, similar threat model), and Numbers Protocol (commercial, blockchain-anchored). WITNESS is more philosophically interesting than any of those and more honest about its limits, but it is a v0.1 single-author artifact and they ship products.

Would I use it? As-is, no — for the same reason the author tells you not to. As a reading exercise, yes, and I'd watch v0.2 (CIGaW peer co-signing + OpenTimestamps anchoring + ephemeral keys) carefully because if those land and a verifier ships, this stops being a demo and starts being something. Effort to make it useful for a real workflow: fix the license file, add a standalone verifier (CLI or static page) so the threat model actually holds end-to-end, rename `index.htmk`, delete or wire up the WebRTC orchestrator stub, and decide whether the proximity-attestation layer is Bluetooth or WebRTC+acoustic before writing more code in either direction.

## Summary

Witness-mobi is a small, single-author reference PWA that does what its landing page says v0.1 does — captures sensor data, triple-hashes it, signs with a non-extractable ECDSA P-256 key, and appends to a chain-linked IndexedDB ledger — implemented impressively in one ~38 KB self-contained HTML file with zero dependencies. The author can code. They are also honest, in print, that this is a demonstration and not something to deploy. The surrounding repo is unfinished in ways that matter: a placeholder-stub LICENSE file that names AGPLv3 without including it, an orphaned WebRTC-orchestrator design doc that contradicts the landing page's Bluetooth roadmap, no verifier, no tests, a typo'd app filename patched at the CDN layer. Read it to learn; don't depend on it; watch v0.2.

---

### Appendix: Investigation Notes

**The shipped PWA (`app/index.htmk`, 812 lines, ~38 KB, 1 inline script):**
- `generateKey({name:'ECDSA',namedCurve:'P-256'}, false, ['sign','verify'])` — non-extractable, as claimed.
- Hand-rolled BLAKE2b (`function compress`, ADD/XOR/ROT/G), Keccak (`function keccakf`); SHA-256 via `crypto.subtle.digest`.
- Real chain verifier: walks `ledgerAll()`, recomputes `sha256(concat(prevHash, body))`, returns `{ok:false, brokenAt:seq}` on mismatch.
- QR built from scratch: `rsGen`, `rsEncode`, `chooseVersion`, `placeFinder`, `setFmt`. No qrcode.js.
- `getUserMedia` wired. Service Worker not present (so "installs to home screen" is the manifest path only — no offline cache logic visible).

**Project hygiene signals:**
- `README.md` is one line: "Reference implementation site for WITNESS — tamper-evident receipts for sensor data".
- `LICENSE.md`: AGPLv3 body literally replaced with `[Full standard AGPLv3 text would be here. Network fetch failed.]`. Dual-license clause sits above it. Legally and reputationally bad for an integrity-focused project.
- `app/index.htmk` — `.htmk` not `.html`. `vercel.json` works around it with a forced `Content-Type: text/html` header for `/app` and `/app/`.
- No tests, no CI, no package manifest of any kind, no service worker, no separate verifier.

**Orchestrator stub (`Acoustic + WebRTC Orchestrat…`, 7 KB, root, not loaded):**
- Class `CigawAcousticWebRTC` uses `RTCPeerConnection` + `wss://relay.indivisa.net` signaling.
- `transmitAcousticToken`/`receiveAcousticToken` are explicit `// Stub` — no Quiet.js init, no Web Audio code.
- `finalizeReceipt(signature)` is an empty method with a comment.
- Conflicts with landing page roadmap, which describes CIGaW as *Bluetooth* peer co-signing, not WebRTC.

**Marketing/code consistency:**
- "Single HTML file, ~38 KB, zero dependencies, zero cloud calls" — true of `app/index.htmk`.
- "Secure Enclave or StrongBox" — overstated; WebCrypto + `extractable:false` is OS-dependent and not enforceable from JS.
- "Anyone with the QR and the original file can verify the receipt without contacting any server" — true in principle; no shipped verifier to actually do it.

**Commit log:**
- 8 commits, 2026-05-13 → 2026-05-17. Single author. The 2026 dates may be a local clock setting; flag for the curious.
