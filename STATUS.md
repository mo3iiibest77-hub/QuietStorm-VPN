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
----:

Scanner (Go — SenPaiScanner fork)

gomobile Go 1.26 compatibility fix applied in android/build_go_mobile.sh: GOFLAGS="${GOFLAGS:-} -ldflags=-checklinkname=0"

Commit: 9a628ce

Android Scanner build succeeded and the APK was installed/tested on the phone.

E2E Android test confirmed: scanner runs successfully, performs the scan, and discovers Cloudflare IPs.

Previous Android release artifact contained only the 3 APK variants; the generated senpaiscanner.aar was not published separately.


Android Client (PattNG fork — QuietStormNG)

CoreOutboundBuilder.kt fingerprint fix committed as 19a7262e: fingerprint = "unsafe" is now hardcoded and overrides config/UI values such as "chrome".

TESTED: the UI may still display Chrome, but runtime uses unsafe.

TESTED: Cloudflare config connects successfully with production fingerprint, cipherSuites, and finalMask values; upload and download work correctly.


Build / Artifact Pipeline

SenPaiScanner Android workflow updated in commit 87c2076.

Workflow now verifies android/app/libs/senpaiscanner.aar.

AAR is copied into the release artifact as: SenPaiScanner-${VERSION}-mobile.aar

Next build must confirm that the AAR is actually present in the downloaded android-release artifact.


OPEN — NEXT STEPS (in order)

1. NEXT: Run a new SenPaiScanner Android build from commit 87c2076

Confirm senpaiscanner.aar is included in the android-release artifact

Transfer the AAR into PattNG/V2rayNG/app/libs/

Wire the Kotlin bridge to the AAR

Do NOT start the new UI or scoring yet



2. Add "Optimize for Cloudflare" button to UI


3. Scoring/ranking in scanner


4. UI theme: black/gold (brand assets exist, see project owner)


5. Post-MVP: scoring persistence, multi-ISP testing


---

## 14) V2rayNG / PattNG Android Build Investigation — Latest Findings

### Current Build Environment

The Android project is located at:

`~/quietstorm-vpn/pattng/V2rayNG`

The project uses:

- Gradle `9.5.1`
- Java `21.0.11`
- OpenJDK `21` on `aarch64`
- Gradle daemon JVM argument: `-Xmx4096m`
- Gradle uses `6` worker leases during the observed build
- Android Gradle Plugin is configured through `gradle/libs.versions.toml`

### Important Build Failure History

The first observed failures appeared during Gradle configuration and initially included:

`The current JVM process isn't compatible with build requirement. The maximum heap size is insufficient.`

Gradle then successfully started a single-use daemon with:

`-Xmx4096m`

Therefore, the currently observed blocking failure is NOT simply an insufficient Gradle heap setting.

The build subsequently progressed through project loading and reached root project configuration.

The current reproducible failure is:

`Plugin [id: 'com.android.application', version: '9.3.1'] was not found`

Gradle searched:

- Google
- Maven Central
- Gradle Plugin Portal

and attempted to resolve:

`com.android.application:com.android.application.gradle.plugin:9.3.1`

The relevant project configuration is:

`build.gradle.kts`:
`plugins { alias(libs.plugins.android.application) apply false ... }`

`gradle/libs.versions.toml`:
`agp = "9.3.1"`

and:

`android-application = { id = "com.android.application", version.ref = "agp" }`

The `settings.gradle.kts` plugin repositories are already configured with:

`google()`
`mavenCentral()`
`gradlePluginPortal()`

The Google repository also has Android/Google/AndroidX content filters.

### Important Diagnostic Correction

A direct HTTP test was initially performed against:

`https://dl.google.com/dl/android/maven2/com/android/tools/build/gradle/<version>/gradle-<version>.pom`

for several AGP versions.

Those tests returned HTTP `404`.

This test was later identified as an invalid/inconclusive diagnostic because the requested artifact path/name does not correctly represent the plugin marker resolution used by Gradle.

Therefore:

**Do NOT record the HTTP 404 results as proof that AGP versions are unavailable from Google Maven.**

The correct next diagnostic must inspect Google Maven metadata and/or the actual Gradle-resolved artifact coordinates.

### Current Known Project Configuration

`gradle/libs.versions.toml` currently contains:

`agp = "9.3.1"`
`kotlin = "2.4.10"`

and the Android application/library plugins both reference the same `agp` version.

`gradle.properties` currently contains:

`org.gradle.jvmargs=-Xmx4096m -Dfile.encoding=UTF-8`

No project file has yet been changed as part of this build investigation.

### Current Root Cause Status

The build is currently blocked during Gradle plugin resolution, before Android compilation begins.

The confirmed failure is:

`com.android.application` version `9.3.1` cannot currently be resolved by the Gradle build from the configured plugin repositories.

The exact reason is NOT yet confirmed.

Possible causes that must be distinguished before changing versions include:

- incorrect/nonexistent AGP version in the Version Catalog;
- repository metadata/version availability issue;
- network/proxy/mirror behaviour;
- Gradle plugin marker resolution problem;
- compatibility between the selected Gradle version and the selected AGP version;
- a project configuration/version combination that was copied from a newer upstream state but is not currently resolvable in this environment.

No AGP downgrade should be performed blindly.

### Build Investigation Commands Already Executed

The following build command was executed:

`./gradlew --no-daemon --stacktrace --info assembleRelease 2>&1 | tee /tmp/pattng-build.log`

It successfully reached:

`Settings evaluated`
`Projects loaded`
`Root project 'v2rayNG'`
`Included projects: [root project 'v2rayNG', project ':app']`

and then failed while evaluating the root build script because the Android application plugin could not be resolved.

### Operational Rule for Future Investigation

Before changing AGP, Kotlin, Gradle wrapper, repositories, or project source files:

1. Confirm the exact artifact/version availability using the correct Google Maven metadata or Gradle resolution mechanism.
2. Confirm the compatibility relationship between Gradle `9.5.1`, AGP, and Kotlin.
3. Preserve the current working tree and avoid unrelated Android source changes.
4. Re-run the smallest diagnostic command necessary before attempting another full `assembleRelease`.
5. Do not treat the earlier HTTP `404` artifact-path test as authoritative evidence.

### Latest State

The project has progressed past the earlier generic daemon/heap failure.

The active blocker is now specifically Android Gradle Plugin resolution for:

`com.android.application:9.3.1`

The exact root cause remains unconfirmed and requires correct repository/artifact metadata inspection before any version change.

---
