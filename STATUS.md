# PROJECT STATUS — Read After PROJECT_FOUNDATION.md

> This file tracks where the project actually stands right now. Update it
> at the end of any work session (human or AI) that changes what's true
> about the codebase. Keep entries factual and dated — this is a status
> log, not a plan. Read `PROJECT_FOUNDATION.md` first for the unchanging
> rules and architecture; this file tells you what's been confirmed,
> what's missing, and what to do next.

---

## Last updated: 2026-08-17

## Repo layout (confirmed on server, /opt/quietstorm-vpn)

```
/opt/quietstorm-vpn/
  pattng/          — clone of patterniha/PattNG, checked out at tag 2.3.4-P22
                     (detached HEAD — no local branch created yet)
  senpai-scanner/  — clone of MatinSenPai/SenPaiScanner, main @ v1.0.0
```

Neither `pattng/` nor `senpai-scanner/` has been modified yet. No commits
have been made in either. `/opt/quietstorm-vpn` itself is not a git repo —
the two clones remain separate repositories, as intended.

---

## Confirmed facts about the codebase (verified by reading actual files, not assumed)

### senpai-scanner/internal/xraytest — real Xray validation layer EXISTS

- `parser.go`: parses `vless://` / `trojan://` / `vmess://` share links into
  `VLESSConfig`. Has `WithAddress()` / `WithEndpoint()` for swapping in a
  candidate IP without touching the rest of the profile. Has
  `Phase2SanityError()` for catching bad WS paths.
- `builder.go`: builds a real xray-core JSON config from `VLESSConfig`
  (SOCKS inbound + vless/trojan/vmess outbound, ws/grpc/xhttp transports).
  **`buildStreamSettings` currently only sets `serverName`, `fingerprint`,
  and `alpn` under `tlsSettings` — no fragment/finalmask layer, no custom
  `cipherSuites` list. This is a confirmed gap, see "Open gaps" below.**
- `runner.go`: actually spins up an `xcore.Instance`, waits for the local
  SOCKS port, and validates through real traffic — connectivity check via
  `/cdn-cgi/trace` (requires `colo=` in body), download throughput, optional
  upload throughput, one automatic retry on failure. `ProxyConnectivityCheck`
  is exported "for the Android gomobile bridge" — **not yet confirmed
  whether that bridge actually exists on the PattNG side.**

### senpai-scanner/internal/prober — screening layer EXISTS

- TCP / TLS / HTTP probe modes, SNI rotation across known Cloudflare
  hostnames, colo detection via CF-Ray and trace body, optional WebSocket
  probe (idle-hold + upgrade check to detect DPI killing long-lived TLS),
  optional "stability" idle-hold check after a successful HTTP probe (Iran
  DPI-specific: catches DPI that allows the initial GET but RSTs shortly
  after).

### senpai-scanner/internal/engine — orchestration EXISTS

- Standard worker-pool pattern (`Run` for streaming IPs, `RunList` for fixed
  lists with a raised timeout floor). Nothing unusual; extend, don't
  rewrite.

### NOT yet found / not yet confirmed

- No scoring/ranking logic located yet.
- No code combining a `prober.Result` (screening) with a
  `xraytest.ValidationResult` (real validation) into one final decision.
- No result persistence layer located yet.
- No CLI subcommand wiring for the xraytest layer located yet (need to check
  `cmd/`).
- PattNG-side Xray/subscription integration not yet inspected (need to look
  at `AngConfigManager`, `V2rayConfig`, `SubscriptionUpdater`, etc.)
- Whether a gomobile bridge already exists on the PattNG/Android side to
  call `ProxyConnectivityCheck` — unconfirmed.

---

## CONFIRMED CORRECTION: production transport is TLS+WS+fragment, not Reality

Earlier working assumption (now retracted) was that this project targets
Reality. The user provided the actual production subscription config and
confirmed:

- Transport: `security: tls`, `network: ws` — **not** `reality`.
- A fragment/obfuscation layer (called `finalmask` in the real config,
  `type: fragment`) that splits the TLS ClientHello and follow-up packets
  per specific `delays` / `lengths` / `maxSplit` values, to evade DPI.
- A custom, specific `cipherSuites` list (not the Go/xray default).
- `fingerprint: "unsafe"`.

**Confirmed required flow:** subscription link received by client → client
applies fragment/finalmask + cipherSuites + fingerprint=unsafe to the parsed
config → scanner-validated candidate IP is swapped into that config → real
connection test runs against that exact final config → VPN connects.

**These three settings must be configurable/overridable in code — but the
DEFAULT, when nothing else is specified, MUST be exactly the values below.
Do not treat these as placeholders or examples; they are the confirmed
production values as of 2026-08-17.**

Default fragment (`finalmask`, applied under `streamSettings`, alongside
`tlsSettings`, as a `tcp` array with two fragment entries):
```json
"finalmask": {
  "tcp": [
    {
      "type": "fragment",
      "settings": {
        "packets": "tlshello",
        "lengths": ["5", "94", "1"],
        "delays": ["0"],
        "maxSplit": "0"
      }
    },
    {
      "type": "fragment",
      "settings": {
        "packets": "1-1",
        "lengths": ["109", "1"],
        "delays": ["1"],
        "maxSplit": "355"
      }
    }
  ]
}
```

Default `cipherSuites` (single colon-joined string, under `tlsSettings`):
```
TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA:TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA:TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256:TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
```

Default `fingerprint` (under `tlsSettings`):
```
unsafe
```

**Design requirement:** when extending `VLESSConfig`/`builder.go` to support
these, add fields with these three exact values as the hardcoded defaults
(e.g. `DefaultFragment`, `DefaultCipherSuites`, `DefaultFingerprint` — naming
is an implementation detail, the defaulting behavior is not). Callers must
be able to override them (e.g. for future support of a different production
config), but if a `VLESSConfig` doesn't specify otherwise, the pipeline must
silently apply exactly the values above — not omit them, not use xray-core's
built-in defaults instead.

**Action needed:** `internal/xraytest/builder.go`'s `buildStreamSettings`
must be extended to inject the fragment/finalmask layer, the custom
cipherSuites list, and `fingerprint: "unsafe"` (all three defaulting to the
exact values above) — otherwise xraytest is validating a different
(simpler) config than what real users actually connect with, making its
pass/fail results not representative of real-world usability. This is the
single most important known next step for making validation results
trustworthy.

---

## NEXT STEPS (in order)

1. Confirm whether `cfg.Fingerprint` already carries `"unsafe"` correctly
   through `ParseVLESS`/`ParseTrojan`/`ParseVMess`, or whether it needs to
   default to `"unsafe"` independently when not present in the share link.
2. Extend `buildStreamSettings` in `builder.go` to add the fragment/
   finalmask layer and cipherSuites list, using the exact default values
   confirmed above (fragment JSON, cipherSuites string, fingerprint=unsafe).
   Make all three overridable, but default to exactly these — values are
   already confirmed, no need to re-ask the user.
3. Re-verify end to end: build a VLESSConfig from a real subscription
   entry, swap in a scanner-confirmed candidate IP, run it through
   `ValidateConfig`, confirm it reports success against the real DPI
   environment (Iranian ISP), not just from the scanner server.
4. Only after step 3 works reliably: start on scoring/ranking and result
   persistence.
5. Only after the scanner side is solid: move to PattNG-side integration
   (how the client receives/uses scanner-validated candidate lists).

Do not skip ahead to scoring, PattNG integration, or branding work before
step 2–3 are done — an unrepresentative validation layer makes everything
built on top of it unreliable.
