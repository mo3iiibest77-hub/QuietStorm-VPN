PROJECT_FOUNDATION.md
# QuietStorm-VPN — Project Foundation

> This file defines the permanent architecture, rules, and reasoning of the
> project. It does not change unless the architecture changes. Read this
> before reading STATUS.md.

---

## What This Project Is

QuietStorm-VPN is a personal VPN client for Iranian users. Its goal is to
provide reliable, upload-capable VPN connections on Iranian ISPs (Irancell,
MCI, etc.) where DPI aggressively blocks upstream traffic on standard
Cloudflare CDN IPs.

The project consists of two active components:

**PattNG** — Android VPN client, forked from patterniha/PattNG (itself a
fork of 2dust/v2rayNG). All client-side code lives here. The app is built
via GitHub Actions on push to master and released as a signed APK.

**QuietStorm-VPN** — This documentation repository only. No source code
lives here. It tracks project state and hands off context between AI
sessions.

There is no longer a separate Go scanner binary (SenPaiScanner). The
Cloudflare IP scanner is now embedded directly inside PattNG as Kotlin
code.

---

## Why Upload Is the Core Problem

Iranian DPI allows download traffic through Cloudflare CDN IPs but blocks
or severely throttles upload. A VPN that cannot upload is useless for
Telegram file uploads, voice calls, and general browsing.

Standard latency-based IP scanners (ping, TCP handshake) cannot detect
this problem. An IP can pass every TCP/TLS test and still have zero upload
throughput on a real Iranian ISP.

This is why the scanner must test real upstream traffic, not just latency.

---
 نه
## The Production TLS Configuration

These three values are the core of the project. Without them, upload does
not work even on clean IPs. They must never be removed, weakened, or
changed without a specific confirmed reason.

**fingerprint:** `unsafe`

**finalMask (fragment):**
```json
{"tcp":[{"type":"fragment","settings":{"packets":"tlshello","lengths":["5","94","1"],"delays":["0"],"maxSplit":"0"}},{"type":"fragment","settings":{"packets":"1-1","lengths":["109","1"],"delays":["1"],"maxSplit":"355"}}]}
```


**cipherSuites:**

TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA:TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA:TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256:TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256

These are stored as constants in `AppConfig.kt` inside `object AppConfig`
and applied by `CloudflareScanner.applyBestIp()` to every selected profile.

---

## Transport Profile

- Protocol: VLESS
- Transport: WebSocket + TLS
- CDN: Cloudflare (IP is scanned and replaced, SNI/Host remain fixed)
- Security: TLS with the cipherSuites and fingerprint above
- Fragment: finalMask applied to TCP stream to evade DPI on TLS ClientHello

---

## The Embedded Scanner Architecture

The scanner lives inside PattNG at: V2rayNG/app/src/main/java/com/v2ray/ang/senpai/CloudflareScanner.kt
It reads Cloudflare IP ranges from `assets/cf_ranges_v4.txt`, picks 30
random candidates (2 per CIDR), and tests each one through:

1. TCP connectivity check
2. Outbound delay measurement via `CoreNativeManager.measureOutboundDelay`
3. Real upload test via `RealTrafficSpeedTest` (8 MB to speed.cloudflare.com/__up)
4. Real download test via `RealTrafficSpeedTest` (8 MB from speed.cloudflare.com/__down)

An IP passes only if upload exceeds 700 KB/s. The best IP is selected by
highest upload throughput, not lowest latency.

The scanner uses the same production TLS configuration (fingerprint,
finalMask, cipherSuites) that the real VPN connection uses. A scanner
result is meaningless if it tests a different configuration.

`RealTrafficSpeedTest` lives at: V2rayNG/app/src/main/java/com/v2ray/ang/service/RealTrafficSpeedTest.kt

It spins up a temporary Xray CoreController with a SOCKS inbound on a
random port, runs the traffic test through it, then shuts it down.

---

## IP Range Reality

No single Cloudflare IP range works well for all Iranian ISPs. Different
ranges perform differently per ISP and per region. The scanner tests across
all ranges randomly. The upload test is the only reliable discriminator.

Known observations (not guaranteed, network conditions change):
- Range 8.x has produced good results on some ISPs
- Range 45.x often has poor upload despite good download
- Range 104.x, 172.x, 162.x vary by ISP

Do not hardcode ranges. Let the scanner test randomly and let upload be the
gate.

---

## Build System

PattNG is built via GitHub Actions on every push to master. The workflow
produces signed APKs (arm64-v8a, armeabi-v7a, x86, x86_64, universal).
The server at `/opt/quietstorm-vpn/pattng/` is the working clone.

To deploy a change:
1. Edit files on the server
2. `git add`, `git commit`, `git push`
3. GitHub Actions builds and uploads APK artifacts
4. Download and install on test device
5. Test on real Iranian ISP

---

## Permanent Rules

- Never test VPN quality from a Dutch server. The target environment is
  Iranian ISP → Android device → Xray → Cloudflare IP.
- Never remove or weaken finalMask, cipherSuites, or fingerprint.
- Never reintroduce a separate Go scanner binary. The scanner is embedded.
- Never judge an IP by latency alone. Upload is the primary metric.
- Never judge an IP by download alone. Download is almost always fine.
- The scanner must use the same config the real VPN uses.
- Build success does not prove real connectivity.
