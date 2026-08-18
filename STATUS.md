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
/opt/quietstorm-vpn/
  pattng/          — clone of patterniha/PattNG, checked out at tag 2.3.4-P22
                     (detached HEAD — no local branch created yet)
  senpai-scanner/  — clone of MatinSenPai/SenPaiScanner, main @ v1.0.0
                     origin remote changed 2026-08-17 to point at the fork
                     github.com/mo3iiibest77-hub/SenPaiScanner (write access
                     needed to push commits — the upstream MatinSenPai repo
                     is read-only to this account). Fork pushed at commit
                     c8bbee7 on main.

Both `pattng/` and `senpai-scanner/` are separate git repos from the docs
repo (`quietstorm-vpn-docs`), as intended — see PROJECT_FOUNDATION.md for
the reasoning against merging them.

---

## Confirmed facts about the codebase (verified by reading actual files, not assumed)

### senpai-scanner/internal/xraytest — real Xray validation layer EXISTS

- `parser.go`: parses `vless://` / `trojan://` / `vmess://` share links into
  `VLESSConfig`. Has `WithAddress()` / `WithEndpoint()` for swapping in a
  candidate IP without touching the rest of the profile. Has
  `Phase2SanityError()` for catching bad WS paths. Now also has
  `CipherSuites` and `DisableFragment` fields (added 2026-08-17, see below).
- `builder.go`: builds a real xray-core JSON config from `VLESSConfig`
  (SOCKS inbound + vless/trojan/vmess outbound, ws/grpc/xhttp transports).
  Updated 2026-08-17 — see "DONE" items below. No longer missing the
  fragment/cipherSuites/fingerprint layer.
- `runner.go`: actually spins up an `xcore.Instance`, waits for the local
  SOCKS port, and validates through real traffic — connectivity check via
  `/cdn-cgi/trace` (requires `colo=` in body), download throughput, optional
  upload throughput, one automatic retry on failure. `ProxyConnectivityCheck`
  is exported "for the Android gomobile bridge" — not yet confirmed whether
  that bridge actually exists on the PattNG side.

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

- Transport: `security: tls`, `network: ws` — not `reality`.
- A fragment/obfuscation layer (called `finalmask` in the real config,
  `type: fragment`) that splits the TLS ClientHello and follow-up packets
  per specific `delays` / `lengths` / `maxSplit` values, to evade DPI.
- A custom, specific `cipherSuites` list (not the Go/xray default).
- `fingerprint: "unsafe"`.

Default fragment (`finalmask`, applied under `streamSettings`, alongside
`tlsSettings`, as a `tcp` array with two fragment entries):

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

Default `cipherSuites` (single colon-joined string, under `tlsSettings`):

TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA:TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA:TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256:TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256

Default `fingerprint` (under `tlsSettings`): unsafe

---

## NEXT STEPS (in order)

1. DONE (2026-08-17): `buildStreamSettings` now defaults `fingerprint` to
   `DefaultFingerprint` ("unsafe") whenever `cfg.Fingerprint` is empty,
   regardless of what parsing produced.
2. DONE (2026-08-17): `internal/xraytest/parser.go` gained two new
   `VLESSConfig` fields (`CipherSuites`, `DisableFragment`).
   `internal/xraytest/builder.go` was rewritten: `buildStreamSettings` now
   injects `DefaultCipherSuites` and `DefaultFingerprint` when the config
   doesn't override them, and appends the `finalmask` fragment layer
   (`defaultFragmentSettings()`) unless `cfg.DisableFragment` is true.
   Verified with `go build ./...` (Go 1.26.1 installed manually — the
   apt-provided `golang-go` 1.18 is too old for this module, which
   requires `go 1.26.1` per `go.mod`) — build succeeded with no errors.
   Committed to the `senpai-scanner` repo (not the docs repo).
3. OPEN — next step: re-verify end to end. Build a VLESSConfig from a real
   subscription entry, swap in a scanner-confirmed candidate IP, run it
   through `ValidateConfig`, confirm it reports success against the real
   DPI environment (Iranian ISP), not just from the scanner server. A
   successful compile only proves the code is well-typed, not that it
   produces a working tunnel against real DPI — this has NOT been tested
   yet.
4. Only after step 3 works reliably: start on scoring/ranking and result
   persistence.
5. Only after the scanner side is solid: move to PattNG-side integration
   (how the client receives/uses scanner-validated candidate lists).

Do not skip ahead to scoring, PattNG integration, or branding work before
step 3 is done — an unrepresentative validation layer makes everything
built on top of it unreliable.

## Environment note

Go 1.26.1 was installed manually under `/usr/local/go` on the build server
via the official tarball, with `PATH` updated in `~/.bashrc`. The distro's
`apt` Go package (1.18) is too old for this project (go.mod requires
`go 1.26.1`).


QuietStorm-VPN — SenPaiScanner Project Status

Current State

The SenPaiScanner Xray validation path has been successfully repaired and validated against a real VLESS over WebSocket + TLS endpoint.

The original end-to-end validation failed before any network test could begin because the bundled Xray configuration builder generated an invalid Fragment mask configuration. The failure was:

"LengthMin can't be 0"

The issue was traced to an incompatibility between the project's current Xray-core version and the Fragment mask configuration generated by "internal/xraytest/builder.go".

Instead of modifying the production fragment values, the Xray-core dependency was updated to commit "5ca6f4b7d4dc". This resolved the configuration-building failure while preserving the existing production-style validation configuration.

Xray-core Compatibility Fix

The project was upgraded from:

"github.com/xtls/xray-core v1.260327.0"

to:

"github.com/xtls/xray-core v1.260327.1-0.20260728075948-5ca6f4b7d4dc"

The required indirect dependencies were updated automatically through Go module resolution.

The important point is that the production Fragment configuration itself was not weakened or removed to make the test pass. The compatibility problem was solved at the Xray-core dependency level.

Production-Style Validation Configuration

The Xray validation configuration uses the same relevant production parameters that were previously added to the scanner, including Fragment, cipher suites, and TLS fingerprint handling.

The Fragment values were intentionally preserved rather than replaced with artificial values merely to satisfy the parser.

A previous attempt to force the values into explicit ranges such as "5-5", "94-94", and "1-1" was reverted because that would have changed the intended production behavior.

The successful solution was therefore to restore the original builder configuration and use a compatible Xray-core revision.

End-to-End Validation

A real VLESS endpoint was tested through the scanner's internal Xray validation path.

Endpoint characteristics:

VLESS over WebSocket and TLS.

The tested endpoint resolved through Cloudflare and used the supplied SNI, WebSocket path, Host header, TLS fingerprint, and endpoint address.

The validation was performed using the actual Xray engine rather than only testing TCP connectivity or TLS handshake availability.

Five consecutive end-to-end tests were executed.

All five tests returned:

Success: true

Bytes received per successful test:

524,288 bytes

Transport:

WebSocket

Retries:

zero

Observed throughput values were approximately:

328 KB/s

276 KB/s

297 KB/s

55 KB/s

252 KB/s

The observed throughput varied significantly between individual runs, which is expected for a real Internet path and should not be interpreted as a fixed bandwidth guarantee.

Most importantly, every complete Xray validation attempt successfully established the configured proxy path and received the expected test payload.

Full Project Test Suite

After the Xray-core update, the complete Go test suite was executed.

All available package tests passed successfully, including:

desktop

export

ipsrc

prober

result

ui

xraytest

The remaining packages reported no test files, which is expected.

There were no test failures in the final full-suite run.

Important Interpretation

The successful five-run E2E result proves that the repaired Xray validation path can successfully validate this real VLESS + WebSocket + TLS configuration.

It does not prove that every ISP, every Cloudflare route, every IP address, every SNI, or every possible VLESS configuration will behave identically.

The distinction between application environment and network environment is important.

Running the same Xray validation from Termux/Ubuntu on a device connected through a particular ISP tests the actual network path available from that ISP. Running the scanner from another machine, application, server, or ISP may produce different results because routing, filtering, packet loss, CDN selection, TLS interception, and network policy can differ.

Therefore, the current result should be treated as a real successful validation sample, not as universal proof of connectivity.

Current Repository State

The SenPaiScanner repository contains the Xray-core compatibility fix and the corresponding dependency updates.

The temporary E2E baseline directory used during testing was removed after validation.

The repository was cleaned so that temporary E2E artifacts were not left in the scanner source tree.

The scanner repository is currently clean and synchronized with its remote main branch.

Documentation Location

The project status documentation belongs to the QuietStorm-VPN documentation/project repository, not to the SenPaiScanner source repository.

STATUS.md was therefore removed from SenPaiScanner after the documentation was moved to the QuietStorm-VPN project.

The canonical project status document is:

QuietStorm-VPN / STATUS.md

This file should be treated as the primary handoff document for future AI sessions working on the project.

Handoff for the Next AI Session

The Xray validation problem that originally blocked real end-to-end testing has been resolved.

The important sequence of findings is:

The initial E2E test failed while building the Xray configuration because the Fragment mask rejected a zero minimum length.

Changing the production Fragment values was tested but intentionally rejected as the wrong solution.

The original builder configuration was restored.

The Xray-core dependency was upgraded to the compatible revision "5ca6f4b7d4dc".

The project then built and passed its complete Go test suite.

A real VLESS over WebSocket + TLS configuration was subsequently validated through the actual Xray engine.

Five consecutive E2E attempts succeeded.

The next development stage should therefore build on this working validation path rather than modifying the already-validated Xray configuration without a specific reason.

The next logical engineering work is to continue the scanner's scoring and ranking pipeline, including how successful Xray validation results are incorporated into candidate ranking and persistence.

Future changes should preserve the distinction between:

basic network reachability,

TLS/transport probing,

and actual Xray end-to-end proxy validation.

A candidate should not be considered fully validated merely because its IP accepts a TCP connection or completes a TLS handshake. The Xray validation layer is the authoritative final verification for configurations that require actual Xray compatibility.

Current Conclusion

The Xray E2E validation path is operational.

The Fragment compatibility issue has been resolved without weakening the intended production configuration.

The complete Go test suite passes.

Five consecutive real-world Xray E2E validations succeeded.

The repository is clean.

The project is ready to continue with the next stage of scanner scoring, ranking, and result persistence.


