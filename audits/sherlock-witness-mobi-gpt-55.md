# Sherlock Report: witness-mobi

**Target**: https://github.com/pwhoppa/witness-mobi/tree/main  
**Type**: GitHub repo  
**Mode**: Deep  
**Investigated**: 2026-05-18

## Identification

`witness-mobi` is a tiny static web repo for WITNESS, a browser/PWA proof-of-concept for tamper-evident media receipts. The inspected `main` branch was at commit `4f9d188178936f47529dd67593ac36dc2136d2fc`, last committed 2026-05-17. It has 6 non-git files and about 1,156 lines: a landing page (`index.html`), the app (`app/index.htmk`), one separate WebRTC/acoustic sketch (`Acoustic + WebRTC Orchestrat`), `vercel.json`, `README.md`, and `LICENSE.md`. There is no `package.json`, no lockfile, no tests, no CI, and no build system. License posture is confused: `LICENSE.md` says dual AGPLv3/commercial and even contains the placeholder “Full standard AGPLv3 text would be here. Network fetch failed.”, while the landing page claims Apache-2.0 with signed releases.

## Function

The repo claims WITNESS “turns any phone into a cryptographic notary”: capture sensor data, hash it with SHA-256/SHA3-256/BLAKE2b-256, sign it with a hardware-bound ECDSA P-256 key, append it to a local IndexedDB ledger, and produce a QR/JSON receipt. The landing page is unusually honest about scope: it says this proves integrity, not truth, and calls the app a reference implementation, not a product (`index.html:650-665`, `733-755`). It also advertises offline operation, no accounts, and future CIGaW co-witnessing (`index.html:740-745`, `795-807`).

The actual app mostly matches the single-device prototype claim. `app/index.htmk` has buttons for photo/audio/video/import (`119-127`), captures with `navigator.mediaDevices.getUserMedia` and `MediaRecorder` (`635-670`), hashes the blob (`689-694`), signs a JSON payload with a non-extractable WebCrypto ECDSA key (`388-400`, `697-718`), stores receipt summaries in IndexedDB (`407-437`), and exports/copies/shares JSON/QR receipts (`735-798`). The live deployment also responds at `https://witness.mobi/app` with `200 text/html`.

The hard part is correctness, and here the prototype cracks. The custom SHA3-256 implementation fails the standard `abc` test vector: it produced `450de04d...2d3e4ecd`; the expected SHA3-256 is `3a985da7...11431532`. The local ledger also breaks after the first receipt: `ledgerAdd` uses the previous `chainHash` as a hex string (`app/index.htmk:411-415`), while `ledgerVerify` later treats prior hashes as decoded bytes (`428-435`). A two-entry simulation verifies sequence 0 OK and sequence 1 FAIL. CIGaW is not implemented in the app; it is a separate, unimported sketch whose acoustic send/receive methods are explicit stubs and whose `finalizeReceipt` is a no-op (`Acoustic + WebRTC Orchestrat:165-191`).

## Opinion

Good prototype, bad evidence tool. The visual direction and threat-model writing are strong. The author understands the social danger of “cryptographically verified” being mistaken for “true,” and the root page says that plainly. The app is pleasantly small, local-first, and dependency-free. For a pitch/demo, this has shape.

But this should not be used for anything consequential yet. A receipt system lives or dies by boring verification. This repo has no tests, no known-answer vectors, no independent verifier, no repeatable build, and at least two core correctness bugs in the hash and ledger paths. The “hardware-bound key” copy also overstates what WebCrypto promises: `extractable: false` is real, but Secure Enclave/StrongBox backing is browser/platform-dependent and not guaranteed by this code.

The repo also markets slightly past reality. Nearby-phone co-attestation is presented as a thing WITNESS does, but the implementation is a detached WebRTC sketch with acoustic placeholders. Source/license links are `href="#"`, signed releases are claimed but not present, and licensing contradicts itself. That is normal for a very young repo, but not acceptable for a tool that wants evidentiary credibility.

Would I use it? As a design study or prototype to fork, yes. As a source of trustworthy tamper-evident receipts, no. If you want timestamp anchoring, use OpenTimestamps. If you want production mobile evidence capture, look at Guardian Project ProofMode. To make WITNESS useful, replace custom crypto with vetted implementations or exhaustive test vectors, fix ledger byte handling, add an external verifier, settle the license, and either implement CIGaW end-to-end or remove it from current-capability claims.

## Summary

Verdict: promising sketch, not dependable software. `witness-mobi` has a strong concept and unusually sober framing, but its cryptographic receipt path is untested and presently wrong in important places. The single-device capture/sign/export flow exists; the co-witnessing protocol does not. Treat it as an early prototype worth watching or forking, not as evidence infrastructure.

---

### Appendix: Investigation Notes

- **Repo facts**
  - Branch: `main`; HEAD: `4f9d188178936f47529dd67593ac36dc2136d2fc`.
  - Last commit: 2026-05-17, “Add CigawAcousticWebRTC class for WebRTC signaling”.
  - Files: `index.html`, `app/index.htmk`, `Acoustic + WebRTC Orchestrat`, `README.md`, `LICENSE.md`, `vercel.json`.
  - No `package.json`, tests, CI, or lockfile found.

- **Static/live checks**
  - `app/index.htmk` inline script parses OK.
  - `Acoustic + WebRTC Orchestrat` parses OK.
  - `https://witness.mobi/app` returned `200 text/html`; `https://witness.mobi/app/index.htmk` returned `200 application/octet-stream`.

- **Crypto checks**
  - SHA-256 `abc`: matched standard vector.
  - SHA3-256 `abc`: mismatch. App returned `450de04d958489d459d347800a31c1fd3b7554ab15feee53338088582d3e4ecd`; expected `3a985da74fe225b2045c172d6bd390bd855f086e3e9d525b46bfe24511431532`.
  - BLAKE2b-256 `abc`: matched Python `hashlib.blake2b(digest_size=32)`.

- **Ledger bug**
  - `ledgerAdd` uses `all[all.length-1].chainHash` directly as `prevHash` (`app/index.htmk:411-415`), so after the first entry it is a hex string, not 32 bytes.
  - `ledgerVerify` decodes `r.chainHash` to bytes for the next iteration (`app/index.htmk:435`). Add and verify use different byte streams for entries after #0.
  - Local simulation result: sequence 0 verifies; sequence 1 fails.

- **Claim/reality mismatches**
  - Landing page says “hardware-backed key” / Secure Enclave or StrongBox (`index.html:711-714`); code creates non-extractable WebCrypto ECDSA key (`app/index.htmk:393-397`) but cannot guarantee hardware backing.
  - Landing page says nearby phones co-attest (`index.html:744`); implementation has acoustic/WebRTC stubs only (`Acoustic + WebRTC Orchestrat:165-177`) and is not integrated into `app/index.htmk`.
  - Landing page says Apache-2.0 and signed releases (`index.html:876-885`); repo license says dual AGPL/commercial and includes incomplete AGPL text (`LICENSE.md:1-18`).
