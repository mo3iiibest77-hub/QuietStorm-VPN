## STATUS.md
# QuietStorm-VPN — Project Status

> Read PROJECT_FOUNDATION.md first. This file tracks current state only.
> Update at the end of every work session. Keep entries factual and dated.

---

## Last updated: 2026-08-20

---

## Repository Layout
/opt/quietstorm-vpn/
pattng/ — working clone of mo3iiibest77-hub/PattNG (branch: master)
senpai-scanner/ — no longer actively developed (scanner is now in PattNG)

Documentation repo: `mo3iiibest77-hub/QuietStorm-VPN` (this repo, main branch)
Client repo: `mo3iiibest77-hub/PattNG` (master branch, built via GitHub Actions)

---

## What Is Done and Confirmed Working

### Scanner embedded in PattNG — DONE

`CloudflareScanner.kt` reads `cf_ranges_v4.txt` from assets, picks 30
random IPs (2 per CIDR), and tests each one. Scanner is triggered from the
main screen bottom bar via a floating action button.

### Upload-first validation — DONE

`RealTrafficSpeedTest.kt` runs a real 8 MB upload through a temporary Xray
instance using the production TLS config. Minimum threshold: 700 KB/s.
IPs that fail upload are rejected. Best IP is selected by highest upload,
not lowest latency.

### Production config applied at scan time — DONE

`applyBestIp()` in `CloudflareScanner.kt` writes the selected IP to the
active profile and ensures fingerprint, finalMask, and cipherSuites are set
correctly on every apply.

### UI — DONE

- Scanner FAB: blue when idle, green when scanning
- Counter (x/30) shown above the FAB
- Progress dots (30 dots) shown during scan
- Main connection FAB aligned with scanner FAB
- Scan count: 30 IPs (reduced from 60)

### AppConfig constants — DONE

`DEFAULT_FINALMASK`, `DEFAULT_CIPHERSUITES`, `DEFAULT_FINGERPRINT` are
defined inside `object AppConfig` in `AppConfig.kt` and referenced by
both the scanner and the server config UI.

### Build — CONFIRMED WORKING

GitHub Actions builds successfully on push to master. APK installs and
runs on real Android device.

### Real-world upload test results (2026-08-20)

Tested IP `8.39.125.94` (range 8.x) via Xray SOCKS proxy on Termux:

- MCI (همراه اول): ~2.0 MB/s upload ✓
- Irancell (ایرانسل): ~1.2 MB/s upload ✓

Both ISPs pass the 700 KB/s threshold on this IP.

---

## Known Issues / Open Items

### UI does not refresh after scan completes

After scanner applies a new IP, the server list in the main UI does not
update automatically. The user must open the server edit screen and tap
save to see the new IP reflected. The underlying DB is updated correctly —
this is a UI refresh issue only.

### No scan results screen

Currently the scanner picks the best IP automatically with no user input.
There is no screen showing all tested IPs with their upload/download/latency
results. The user cannot choose from candidates manually.

Planned: A dedicated scan results Activity showing each IP with ping,
upload KB/s, download KB/s, and a select button.

### App name and icon not yet changed

The app still shows the default PattNG name and icon. Planned rename:
QuietStorm NG. Icon replacement pending.

### STATUS.md and PROJECT_FOUNDATION.md not yet committed to this repo

These files need to be updated manually by copying this content into the
QuietStorm-VPN repo.

---

## Next Steps (in order)

1. Fix UI refresh after scan (server list should update without manual edit)
2. Build scan results screen (show all IPs with upload/download/latency)
3. Rename app to QuietStorm NG and replace icon
4. Test on more ISPs and regions to gather upload threshold calibration data
5. Update STATUS.md after each session
