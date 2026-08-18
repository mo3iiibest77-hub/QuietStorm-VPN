# PROJECT STATUS — Read After PROJECT_FOUNDATION.md

> This file tracks where the project actually stands right now. Update it
> at the end of any work session (human or AI) that changes what's true
> about the codebase. Keep entries factual and dated. Read
> PROJECT_FOUNDATION.md first for the unchanging rules and architecture;
> this file tells you what's been confirmed, what's missing, and what to
> do next.

---

## Last updated: 2026-08-19

---

## Repo layout (confirmed on server, /opt/quietstorm-vpn)

/opt/quietstorm-vpn/
pattng/ — fork of patterniha/PattNG (fork of 2dust/v2rayNG)
branch: master, our custom commits on top
senpai-scanner/ — fork of MatinSenPai/SenPaiScanner
branch: main @ fork mo3iiibest77-hub/SenPaiScanner
docs/ — this repo (mo3iiibest77-hub/QuietStorm-VPN)
contains PROJECT_FOUNDATION.md and STATUS.md


---

## Android Client (PattNG) — confirmed state

### Build: GREEN ✅
PattNG builds successfully and produces 4 APKs:
- arm64-v8a (~60 MB)
- armeabi-v7a (~61 MB)
- universal (~135 MB)
- x86 (~125 MB)

APK installs and connects to a manually configured VLESS+WS+TLS subscription.
Basic VPN functionality confirmed working on device.

### What was done to reach green build

Attempt 1: Add senpaiscanner.aar to PattNG/V2rayNG/app/libs/
  Result: FAILED — conflict on libgojni.so between senpaiscanner.aar and
  libv2ray.aar (both bundle the full xray-core Go runtime).

Attempt 2: Add SenPaiScanner-1.0.0-mobile.aar (second build variant)
  Result: FAILED — same Go runtime conflict. Two Go runtimes cannot
  coexist in one Android APK.

Root cause: senpaiscanner.aar bundles its own xray-core + libgojni.so.
  PattNG already has libv2ray.aar with the same. Irreconcilable conflict.

Decision: Abandon the .aar approach entirely. Use PattNG's own xray-core
  (already present via libv2ray.aar + CoreNativeManager) to implement
  scanning in pure Kotlin — no second Go runtime needed.

SenPaiScannerBridge.kt was removed (it imported com.matinsenpai.senpaiscanner
  which no longer exists after .aar removal). This was the last build blocker.
  Removed via: git rm ...senpai/SenPaiScannerBridge.kt → build green.

### CloudflareScanner.kt — added ✅
Location: V2rayNG/app/src/main/java/com/v2ray/ang/senpai/CloudflareScanner.kt

This is the Kotlin-native replacement for the .aar bridge. It implements
the full Cloudflare IP scanning pipeline using PattNG's own xray-core:

Flow:
  1. Caller provides a saved profile guid (VLESS+WS+TLS from subscription).
     That profile already has finalMask (fragment), cipherSuites,
     fingerPrint=unsafe, sni, alpn — all production settings.
  2. For each candidate Cloudflare IP:
     a. Clone the ProfileItem, swap only server = candidateIP.
        All TLS/transport/obfuscation settings inherited unchanged.
     b. Save clone under a throw-away guid in MmkvManager.
     c. CoreConfigManager.getV2rayConfig4Speedtest() builds a lightweight
        xray JSON config (same path RealPingWorkerService uses).
     d. CoreNativeManager.measureOutboundDelay() starts a real xray
        instance, sends HTTP through the tunnel, measures latency, tears
        xray down.
     e. Remove the temporary profile from MmkvManager.
  3. Find candidate with lowest latency.
  4. applyBestIp(guid, bestIp) writes the winning IP back onto the
     original profile. Next VPN connect uses it.

Key point: this tests the exact production config shape (with fragment,
  cipherSuites, fingerprint=unsafe) — not a stripped version. A candidate
  that passes is genuinely usable for real users on Iranian ISPs.

Build status: GREEN ✅ (confirmed 2026-08-19)

---

## Scanner (SenPaiScanner / Go) — confirmed state

### Xray E2E validation: WORKING ✅

- internal/xraytest exists and works end-to-end.
- builder.go applies fragment/finalmask, cipherSuites, fingerprint=unsafe
  (fixed 2026-08-17).
- Xray-core upgraded to v1.260327.1-0.20260728075948-5ca6f4b7d4dc to fix
  Fragment "LengthMin can't be 0" error without changing production values.
- 5 consecutive E2E tests against real VLESS+WS+TLS endpoint: all passed.
  Throughput: 55–328 KB/s (variation expected on real internet paths).
- Full Go test suite passes (desktop, export, ipsrc, prober, result, ui,
  xraytest packages).

### What is NOT done yet (scanner side)

- No scoring/ranking logic implemented yet.
- No code combining prober.Result (TCP/TLS screening) with
  xraytest.ValidationResult into one final ranked output.
- No result persistence layer.
- No CLI subcommand wiring for xraytest layer (cmd/ not yet inspected).

---

## Integration between scanner and client — NOT done yet

The scanner (Go, server-side) and the client (Kotlin, Android) are not
yet connected. Two separate approaches exist and are not yet unified:

- Server-side: SenPaiScanner probes IPs from the scanner server (Go).
- Client-side: CloudflareScanner.kt probes IPs from the device (Kotlin).

These serve different purposes (see PROJECT_FOUNDATION.md §5). Both are
valid. The client-side scanner (CloudflareScanner.kt) is what runs on the
user's device and tests from inside their actual network (Iranian ISP) —
this is the most important validation signal.

---

## UI — NOT done yet

Current UI: stock PattNG/v2rayNG UI, unchanged.

Planned:
- Brand identity: QuietStormNet (QSN)
- Color scheme: deep black background + gold (#C9A84C range) + dark smoke
- Logo: provided as design references (black/gold swirl + shield motifs)
- A "Scan" button needs to be added to trigger CloudflareScanner.scan()
- Result display: show scanning progress + best IP found
- Full UI redesign: post-MVP milestone per PROJECT_FOUNDATION.md §8

---

## NEXT STEPS (in order)

1. ADD UI trigger for CloudflareScanner:
   - A "Find Best IP" / "Scan" button in the main screen
   - Calls CloudflareScanner.scan() with the active profile's guid
     and a list of Cloudflare candidate IPs
   - Shows progress (X/total tested)
   - On finish: calls applyBestIp() then connects VPN
   This is the minimum needed to test the scanner end-to-end on a real device.

2. TEST CloudflareScanner on device:
   - Install APK, load a real VLESS+WS+TLS subscription
   - Trigger scan with a list of known Cloudflare IPs
   - Verify: does it find a working IP? Does VPN connect after?
   - Check logcat for CloudflareScanner TAG output

3. UI redesign (QuietStormNet branding):
   - Apply black/gold color scheme
   - Replace app name, package id, icon
   - Integrate scan button into redesigned main screen

4. Scanner server-side (Go): scoring/ranking + result persistence
   Only after client-side scanner is confirmed working.

5. Connect server-side scanner output to client:
   - Decide delivery format (subscription-style list? API endpoint?)
   - Client fetches pre-validated IP list from scanner server
   - CloudflareScanner validates locally from that shortlist

Do not skip step 2 before starting step 3.
A passing compile does not mean the scanner works on a real device.
ENDOFSTATUS
