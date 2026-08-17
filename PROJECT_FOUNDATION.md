# PROJECT FOUNDATION — Read This First

> This document is the entry point for any AI assistant (chatbot, coding agent,
> or plugin with repository access) working on this project. Read this file
> completely before touching any code, proposing changes, or answering
> questions about this repository. If you are an AI assistant reading this
> for the first time, treat it as your operating charter for this project.

---

## 1. WHAT THIS PROJECT IS

This repository combines two previously separate open-source projects into a
single product:

1. **Scanner engine** (Go) — originally based on `SenPaiScanner`
   (https://github.com/MatinSenPai/SenPaiScanner), a Cloudflare IP
   scanner/prober.
2. **Android VPN client** (Kotlin) — originally forked from `v2rayNG`
   (https://github.com/2dust/v2rayNG) via `PattNG`
   (https://github.com/patterniha/PattNG).

The product being built is a **standalone VPN client and companion scanning
service**. It is derived from v2rayNG's codebase but is **not** v2rayNG and
is **not** marketed, packaged, or branded as v2rayNG. Think of it as: same
proven engine underneath, different product identity on top.

**Current phase: MVP.** Do not treat this as a mature, feature-complete
product. Assume incompleteness everywhere except where the code says
otherwise.

---

## 2. THE CORE PROBLEM THIS PROJECT SOLVES

A Cloudflare IP being reachable does not mean it is a good VPN endpoint.
Ping success, TCP success, TLS success, and even a clean `/cdn-cgi/trace`
response are all necessary but *not sufficient* signals. The only real proof
that an IP is usable is that the actual VPN protocol — for this project,
confirmed to be VLESS over TLS+WS with fragment-based obfuscation (see §3.3;
NOT Reality) — successfully connects and passes data through it.

The project's reason to exist is closing that gap: turning a big list of
"technically reachable" Cloudflare IPs into a small list of
*experimentally validated, ranked, production-usable* VPN endpoints — and
delivering those endpoints into a real Android VPN client.

Pipeline, conceptually:

```
DISCOVERY → SCREENING → CLOUDFLARE VALIDATION → PERFORMANCE TEST
→ REAL PROTOCOL (Xray) VALIDATION → SCORING → RANKING → DELIVERY TO CLIENT
```

A fast IP that fails real Xray validation is worthless. A slightly slower IP
that reliably completes the real tunnel and moves data is the target output.

---

## 3. REPOSITORY / COMPONENT MAP

```
/scanner/    Go — the probing & validation engine (based on SenPaiScanner)
/client/     Kotlin/Android — the VPN client (based on PattNG / v2rayNG)
/docs/       Architecture notes, decisions, this file
```

(Adjust this map to match the actual folder layout once finalized — this
section should always reflect reality, not aspiration. If you are an AI
assistant and the actual folder layout differs from what's described here,
say so explicitly rather than silently assuming this doc is correct.)

### 3.1 Scanner engine — responsibilities
- DISCOVER candidate Cloudflare IPs
- PROBE them (TCP / TLS / HTTP layers) — `internal/prober`
- VALIDATE against real Cloudflare behavior (`/cdn-cgi/trace`, colo, CF-Ray)
- MEASURE latency, loss, jitter, throughput
- VALIDATE against the **real target VPN protocol** via an actual xray-core
  instance — `internal/xraytest` (see §3.3 — this already exists, do not
  rebuild it from zero)
- SCORE and RANK candidates — **not yet implemented, see STATUS.md**
- OUTPUT a structured, ranked result set with full diagnostic detail (not
  just pass/fail)

### 3.3 Confirmed existing: `internal/xraytest` (real Xray validation layer)

Contrary to an earlier assumption drawn from the public repo's README/roadmap
(which suggested real Xray validation was unbuilt), this specific checkout
already has a working implementation. Any assistant working on this repo
must read the actual files before assuming a gap exists — do not trust the
public README's roadmap section as a proxy for what this codebase contains.

What exists:
- `parser.go` — parses `vless://`, `trojan://`, `vmess://` share links into a
  `VLESSConfig`, with `WithAddress()` / `WithEndpoint()` to swap in a
  candidate Cloudflare IP while keeping the rest of the profile — this is
  the Candidate-IP-vs-Target-Profile separation called for in §9 of the
  original brief.
- `builder.go` — generates a real xray-core JSON config (SOCKS inbound +
  vless/trojan/vmess outbound) from a `VLESSConfig`, covering ws/grpc/xhttp
  transports and TLS settings.
- `runner.go` — actually starts an `xcore.Instance`, waits for the local
  SOCKS port, and drives real traffic through it: a connectivity check
  (`/cdn-cgi/trace` through the tunnel, requires `colo=` in the body),
  download throughput, optional upload throughput, with one automatic retry.
  `ProxyConnectivityCheck` is exported specifically for an Android gomobile
  bridge — confirm during inspection whether that bridge is actually wired
  up on the PattNG side or only intended.

**CONFIRMED (2026-08-17): the production target transport is TLS + WS, NOT
Reality.** An earlier version of this document speculated about a Reality
handling gap in `buildStreamSettings` — that speculation was wrong and is
retracted. The actual subscription config this project distributes to real
users uses `security: tls` with:
- a fragment/obfuscation layer (referred to as `finalmask` in the real
  config, `type: fragment`, with specific `delays`/`lengths`/`maxSplit`
  values) that splits the TLS ClientHello and subsequent packets to evade
  DPI,
- a custom `cipherSuites` list,
- `fingerprint: "unsafe"`.

**None of these three (fragment/finalmask, cipherSuites, fingerprint=unsafe)
are currently applied anywhere in `internal/xraytest/builder.go`.** This is
a real, confirmed gap: `buildStreamSettings` only sets `serverName`,
`fingerprint` (from `cfg.Fingerprint`, which may or may not carry
`"unsafe"` through parsing — verify), and `alpn` under `tlsSettings`, with
no fragment layer and no custom cipher suite list.

**Required real-world flow (confirmed by the user, do not deviate):**
subscription link received → client applies fragment/finalmask +
cipherSuites + fingerprint=unsafe to the parsed config → scanner-validated
candidate IP is swapped in → real connection test happens against that
exact final config → VPN connects.

Any Xray validation that does NOT include the fragment/cipher/fingerprint
layer is testing a materially different config than what real users
actually connect with, and its pass/fail result cannot be trusted as
representative. This must be fixed before `xraytest` validation results are
treated as meaningful for this project's real deployment.

### 3.2 Client — responsibilities
- IMPORT / STORE subscription and server data
- SELECT and RUN the Xray core with a chosen server
- CONNECT and maintain the VPN tunnel
- Perform its own **client-side, on-device** testing (this is a distinct
  layer from server-side scanning — see §5)

**These two components have separate responsibilities and should not be
merged.** They communicate through a defined data interface (a ranked
candidate list / subscription format), not by sharing internals.

---

## 4. WHAT TO PRESERVE — DO NOT REWRITE FROM SCRATCH

- Do NOT rewrite the scanner's existing worker/concurrency architecture
  (config, stats, probe workers, result callbacks) from zero. Extend it.
- Do NOT rewrite the client's existing subscription, profile, or Xray
  integration architecture from zero. Extend it.
- Do NOT change the Android client's UI/UX as part of this MVP. Rebranding
  (app name, package id, icon) is in scope; a UI redesign is explicitly
  **out of scope** until the MVP's core validation pipeline works.
- Do NOT remove existing functionality without a concrete, stated reason.
- Do NOT introduce new external infrastructure/dependencies unless the
  existing architecture genuinely requires it.

Before modifying anything: inspect current architecture → understand
existing behavior → identify the smallest required change → implement
incrementally → verify after each change. No giant rewrites.

---

## 5. IMPORTANT DISTINCTIONS THE ASSISTANT MUST NOT COLLAPSE

- **Generic Cloudflare edge health** (an IP looks like a working Cloudflare
  edge) is a *different question* from **actual VPN endpoint usability**
  (this IP works with our specific VLESS+TLS+WS+fragment configuration —
  see §3.3). Passing the first does not imply the second. Real Xray
  validation, run against the exact production config shape including the
  fragment/cipher/fingerprint layer, is mandatory before a candidate is
  considered production-worthy.
- **Server-side probing** (from the scanner's own server) is *not* the same
  as **client-like / real-network probing** (from an actual restricted
  mobile network in Iran). "Works from the scanner server" ≠ "works for the
  end user." Be explicit about which kind of measurement any result
  represents. Do not claim universal usability from one server's
  measurements alone.
- A single successful probe does not establish stability. Prefer a
  candidate that is slightly slower but consistently works over one that is
  fast but unreliable. Repeated validation exists for this reason.

---

## 6. FAILURE HANDLING PRINCIPLE

Never collapse failures into a single generic error. Distinguish causes
(TCP timeout vs TCP refused, TLS handshake failure vs certificate failure,
Cloudflare-not-detected vs colo-unavailable, Xray startup failure vs
handshake failure vs data-transfer failure, etc.). Every result must carry
enough diagnostic detail to explain **why** it passed or failed — a bare
`healthy=true/false` is not acceptable output.

---

## 7. HOW AN AI ASSISTANT SHOULD WORK IN THIS REPO

1. Read this file in full before proposing changes.
2. Inspect the actual current code before assuming what exists — this
   document describes intent and constraints, not a guaranteed up-to-date
   inventory of the code.
3. State explicitly: what already exists, what's missing, what can be
   reused, what must be extended, what must not be touched.
4. Propose the smallest coherent next step, not a full rewrite plan.
5. Implement incrementally; after each meaningful change, verify it builds
   and behaves as expected before moving on.
6. If a request conflicts with something in this document (e.g., "just
   rewrite the whole scanner"), flag the conflict explicitly instead of
   silently complying or silently refusing.
7. If the assistant is uncertain about repo layout, naming, or current
   state, say so rather than inventing details.

---

## 8. PRODUCT IDENTITY (branding — separate from engineering scope)

- This is a standalone client, distinct from v2rayNG in name, package id,
  and icon. It is **not** presented to end users as v2rayNG or as a
  modified v2rayNG.
- Underlying engine and architecture are reused deliberately — this is a
  pragmatic engineering choice, not a UI or brand decision, and should not
  be treated as one.
- Full custom UI/UX is a deliberate **post-MVP** milestone, not part of the
  current scope. Do not let branding work block or dilute effort on the
  validation pipeline, which is the actual product value.

---

## 9. DEFINITION OF MVP SUCCESS

Given a list of candidate Cloudflare IPs and a real target VPN
configuration, the pipeline should produce a ranked output where:

- dead IPs are eliminated
- non-Cloudflare IPs are eliminated
- unusable TLS/SNI candidates are eliminated
- poor-performing candidates are deprioritized
- transport-incompatible candidates are eliminated
- candidates that fail real Xray validation are eliminated
- genuinely working candidates are ranked highest
- every result carries enough diagnostic detail to explain why it passed or
  failed

The deliverable is not "here are some Cloudflare IPs." It is: "here are the
Cloudflare edge IPs experimentally validated against this specific VPN
configuration and ranked by actual, evidenced usability."
