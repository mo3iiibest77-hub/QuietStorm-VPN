# PROJECT STATUS — Read After PROJECT_FOUNDATION.md

> Update this file at the end of any work session. Read PROJECT_FOUNDATION.md
> first for unchanging rules; this file tells you what's confirmed, what's
> missing, and what to do next.

---

## Last updated: 2026-08-18

## Repo layout

Docs repo: https://github.com/mo3iiibest77-hub/QuietStorm-VPN
Scanner fork: https://github.com/mo3iiibest77-hub/SenPaiScanner
Android client fork: https://github.com/mo3iiibest77-hub/PattNG

Local on build server (/opt/quietstorm-vpn/):
  senpai-scanner/  — Go scanner, main @ c8bbee7+
  pattng/          — Android client, master branch

Local on phone (Termux/Ubuntu proot, ~/quietstorm-vpn/):
  senpai-scanner/  — same fork
  pattng/          — same fork
  docs/            — QuietStorm-VPN docs repo

---

## CONFIRMED DONE

### Scanner (Go — SenPaiScanner fork)
- Real Xray validation layer exists and works (internal/xraytest)
- parser.go: parses vless/trojan/vmess share links, WithAddress() for IP swap
- builder.go: extended with DefaultCipherSuites, DefaultFingerprint="unsafe",
  defaultFragmentSettings() (finalMask with two TCP fragment stages)
- runner.go: spins up real xcore.Instance, validates through real traffic
- E2E test: 5/5 success on real Iranian ISP (Termux, no VPN), real data passed
- go.mod xray-core updated to newer commit to fix LengthMin=0 error with
  production fragment values
- Build: go build ./... passes with Go 1.26.1

### Android Client (PattNG fork — QuietStormNG)
- App name: QuietStormNG
- Package name: com.quietstorm.ng (no conflict with v2rayNG)
- GitHub Actions CI: working, builds 4 APK variants (arm64, armeabi, x86,
  universal), ~60MB
- Keystore: alias=quietstorm, storepass/keypass=QuietStorm2026
- CoreOutboundBuilder.kt patched:
  * fingerprint = "unsafe" (hardcoded, overrides any config value including
    "chrome" — this was the key fix; ?: "unsafe" was not enough because
    profileItem.fingerPrint="chrome" was non-empty)
  * cipherSuites defaults to full production list when config value is empty
  * finalMask defaults to production fragment (tlshello + 1-1) when config
    has no finalMask
- TESTED: config imported → VPN connected → upload+download working on
  Iranian ISP through Cloudflare CDN. Confirmed that fingerprint must be
  "unsafe" (not "chrome") for connection to succeed.

---

## PRODUCTION CONFIG DEFAULTS (do not change without testing)

fingerprint: "unsafe" (hardcoded in CoreOutboundBuilder.kt)

cipherSuites default:
TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA:TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA:TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256:TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256

finalMask default (TCP fragment):
{"tcp":[{"type":"fragment","settings":{"packets":"tlshello","lengths":["5","94","1"],"delays":["0"],"maxSplit":"0"}},{"type":"fragment","settings":{"packets":"1-1","lengths":["109","1"],"delays":["1"],"maxSplit":"355"}}]}

---

## OPEN — NEXT STEPS (in order)

1. NEXT: Build scanner into Android client via gomobile bridge
   - ProxyConnectivityCheck already exported in runner.go for this purpose
   - Need to: compile Go scanner as Android library (.aar) using gomobile
   - Wire up to Android: user presses "Optimize for Cloudflare" button
   - Scanner runs on-device with real ISP, finds best Cloudflare IPs
   - Best IP replaces config address, VPN connects

2. Add "Optimize for Cloudflare" button to UI
   - One button that triggers: inject defaults → scan IPs → connect

3. Scoring/ranking in scanner
   - Combine prober.Result (screening) with xraytest.ValidationResult
   - Score by: reliability + latency + throughput + CF validation

4. UI theme: black/gold (brand assets exist, see project owner)

5. Post-MVP: scoring persistence, multi-ISP testing

---

## IMPORTANT NOTES

- E2E validation MUST run on actual Iranian ISP (Termux on phone, no VPN),
  NOT from the Dutch server (NDL) — server has open internet, results are
  not representative of real DPI environment
- fingerprint must be "unsafe" not "chrome" or any other value for
  Cloudflare WS+TLS configs to connect on Iranian ISPs
- Scanner server-side role: build artifacts only. All validation runs
  client-side on phone.
- Go 1.26.1 installed manually at /usr/local/go (apt version 1.18 too old)
