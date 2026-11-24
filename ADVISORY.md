# Security advisory: XCPE backdoor preinstalled in MECOOL KM2 Plus firmware (re-shipped in an OTA after the takedown; removed in C1.3.5 without rotating the platform key)

**Author:** Javier Aguilar ([@javier3822007](https://github.com/javier3822007)), Chile
**Status:** public. Payload removed in `C1.3.5_20261111` (Nov 2026); platform key not rotated (see [Fixed build and residual exposure](#fixed-build-and-residual-exposure-update-2026-11-23))
**Last update:** 2026-11-23
**Full notes (Spanish, more detail):** [README.md](README.md)

---

## Summary

The official firmware of the MECOOL KM2 Plus Android TV box includes the XCPE backdoor (package `com.oemservice`, internal name `x.cpe`). It is preinstalled as a privileged system app, signed with the firmware platform key, and runs as `android.uid.system`. The full payload is stored in the `/oem` partition and the stub in `/system` installs it on boot.

I found it in my own two devices, on three different firmware versions. The most important part for me: **firmware C1.3.3, delivered by OTA to one of my devices between 6 and 8 November 2026, still contained exactly the same backdoor** (same SHA-256). This was after Google published the takedown (2 Nov 2026) and after MECOOL told me by email that the existing products "can continue to be used normally and safely". A later build, C1.3.5 (published 13 Nov 2026), removes the backdoor but keeps the same platform key.

I am not a professional security researcher, this was done on my own devices for defensive reasons. Everything here is static analysis plus `adb` checks on the devices. I never saw the backdoor receive a command, because when I started the C2 domains were already down.

## Affected product

| Field | Value |
|---|---|
| Brand / model | MECOOL KM2 Plus (`ro.mecool.device.model=KM2PLUS`) |
| Product / device | `HP4414-MECOOL` / `YYC` |
| `ro.product.manufacturer` | `SkyworthDigital` |
| Android | 14 |
| ODM (according to Google) | Videostrong Technology Co., Ltd. |

**Firmware versions confirmed affected (on my devices):**

| Firmware (`ro.build.fingerprint` build ID) | `oem.buildnumber` | XCPE in `/oem` | How I got it |
|---|---|---|---|
| `C1.2.7_20250822` | `C1.2.7_20250822` | 6.8.4 | device B, current firmware |
| `C1.3.1_20260314` | `C1.3.1_20260314` | 6.9.2 | device A, until 6 Nov 2026 |
| `C1.3.3_20260611` | `C1.3.3_20260615` | 6.9.2 (identical file) | device A, OTA installed 6–8 Nov 2026 |

On 10 Nov 2026 C1.3.3 was the newest version offered to my devices. `C1.3.5_20261111`, published on 13 Nov 2026, is **not** affected (payload removed; see below). Other MECOOL models and other brands built by the same ODM are very probably affected too, but I can only confirm what I have in my hands.

## Vulnerable component

| Path | Version | SHA-256 |
|---|---|---|
| `/system/priv-app/app_xcpe_preset/app_xcpe_preset.apk` (stub, `com.oemservice`) | 1.2.1 | `88e1b6ec984534def6e05421d973e7bf45fe602d929bd3edbb5aaa7e0a3edb7f` |
| `/oem/app_xcpe_ext_sign` (full payload, C1.2.7) | 6.8.4 | `c9baee088d95f24ea9945ba6f0f9f9d717680aaa4d347a42e912785338ac539f` |
| `/oem/app_xcpe_ext_sign` (full payload, C1.3.1 and C1.3.3) | 6.9.2 | `aa6269b993a6ca4f4286ed20cd6b3d7ae7ac4edc329d10ed525c602b60c4fbe5` |
| `/data/app/.../com.oemservice-.../base.apk` (running on device B when I found it) | 6.5.0.20230804 | `2a443baa14be4a2a45b7ea477a8ffc326169aaecc89919a42cf386d0978db0f2` |

None of these hashes are in the IoC list of the Google report, so they are additional samples.

**Signing certificate** (same for stub, `/oem` payload and the running version):

```
CN=skydigital, OU=SKYWORTH, O=SKYWORTH, L=ShenZhen, ST=GuangDong, C=CN, E=skydigital@skyworth.com
SHA-1  1C:6A:0B:B5:70:10:A7:7C:BF:29:A4:95:FF:55:F7:36:2F:E3:32:A3
Serial 3a4085b2e52b00573f11f380eac3da2664f6cd75
```

This is the platform key of the firmware (it also signs the `android` package and SystemUI). So the backdoor is part of the official signed build, it is not something added later by a third party.

## Impact

- **CWE-506** (Embedded Malicious Code) and **CWE-912** (Hidden Functionality).
- The app runs as `android.uid.system` and has, among others, `MASTER_CLEAR`, `REBOOT`, `INJECT_EVENTS`, `READ_FRAME_BUFFER`, `WRITE_SECURE_SETTINGS`, `INSTALL_PACKAGE_UPDATES`, `DELETE_PACKAGES`.
- From the code (strings, manifests, MQTT topics) it can: run shell commands and scripts sent by the operator, silently install/uninstall apps, disable Play Protect package verification, capture the screen, inject key events, upload logs and the list of installed apps, lock or factory reset the device remotely, and (6.5.0) act as a proxy relay.
- The user is not informed and cannot remove it from the settings.
- The C2 domains were taken down in November 2026, but the implant is still in every firmware before C1.3.5 and still trusts its bootstrap domains (`*.anyevonline.com`, `*.szbboys.com`). In my opinion, whoever controls these domains in the future controls the devices again. I did not test this.

On device B the 6.5.0 version was **active, with its MQTT client running**, on a box in normal daily use since 2024.

## The November 2026 OTA

Device A was on C1.3.1 on 6 Nov 2026 (I have it written in my notes) and on C1.3.3 on 8 Nov 2026. The update was installed silently on reboot (A/B device, `sys.boot.reason=reboot,ota`). I had `com.sdt.ota` removed for user 0 at that time, so the OTA did not come through the OEM app. **Correction (2026-11-23):** on this device firmware OTAs are delivered by Google System Update (GMS + `update_engine`) from `android.googleapis.com`. My earlier guess (`com.sdt.upgrade` / `SdtUpgradeService`) was wrong: that service is benign and only migrates display/HDR settings after an OTA.

- `oem` is listed in `ro.product.ab_ota_partitions`. The OTA rewrote the whole `/oem` partition (all files now dated 2026-06-15) and wrote the **same XCPE 6.9.2 again, byte by byte**.
- The stub in `/system/priv-app/app_xcpe_preset` is also identical.
- So the backdoor was not "forgotten" from an old build. It is still being packaged and distributed.

## Fixed build and residual exposure (update 2026-11-23)

**Fixed build.** `C1.3.5_20261111` was published on Google's OTA servers on 13 Nov 2026 as a full A/B OTA with `ota-wipe=yes`. It **removes XCPE**:
- `/oem` no longer contains `app_xcpe_ext_sign` or `BoxFactoryTest_sign`.
- A sweep of 3314 files found none of the indicators of this advisory (domains, AES key, `x.cpe` classes, paths).
- Device A was updated to it on 21 Nov 2026 and factory-reset itself, as the package requires.

The following risks are **not** resolved by that build:

1. **The platform key was not rotated.** `framework-res.apk` (the `android` package) and the platform-signed system apps of `C1.3.5_20261111` are signed with **the same certificate** that signed every XCPE sample in this advisory:
   ```
   CN=skydigital, OU=SKYWORTH, O=SKYWORTH, L=ShenZhen, ST=GuangDong, C=CN, E=skydigital@skyworth.com
   SHA-1    1C:6A:0B:B5:70:10:A7:7C:BF:29:A4:95:FF:55:F7:36:2F:E3:32:A3
   SHA-256  a5632097dbedf19c5972868d8ea85926480b9f25cb5f6393a2b11c6af10a613d
   Serial   3a4085b2e52b00573f11f380eac3da2664f6cd75
   ```
   On a "fixed" device, any APK signed with this key that declares `android:sharedUserId="android.uid.system"` still gets system privileges once it is installed. That includes existing or future XCPE builds. Whoever signed XCPE can still ship system-level code to these devices. I have not checked whether other brands' firmware uses this key.
2. **The infected firmware is still distributed.** The official `C1.2.7_20250822` package, which carries XCPE 6.8.4 in `/oem`, is still served by Google's OTA infrastructure:
   - URL: `https://android.googleapis.com/packages/ota-api/package/85b6685798448bac95049c6a2ce39ccf3a2618bb.zip`
   - SHA-256: `52e2f0e759970e1d5cddad3090b7a4314023a902b87faf7450d9dc65ce8e4f1c`
   - Status on 2026-11-23: HTTP 200, 971,216,553 bytes.
   - I did not find a package URL for C1.3.3, so I make no claim about it.
3. **Devices that have not updated remain infected.** On C1.2.7, C1.3.1 or C1.3.3, XCPE stays in `/oem` and the stub installs it on boot. Hiding it with `pm uninstall --user 0` survived reboots and an OTA in my tests, but a factory reset undoes that. Those devices keep trusting the bootstrap domains (`*.anyevonline.com`, `*.szbboys.com`). The fix also requires a full wipe, which users may postpone. Device B is still on C1.2.7 as of this update.
4. **OEM apps in the fixed build (and earlier ones) expose unprotected components** (static analysis, not tested on a device). These are separate from XCPE and only listed here for completeness:
   - `com.sdt.ota`: `BootService` is exported without a permission.
   - `com.sdt.producttest` (system UID): `PropertiesProvider` is exported without a permission.

**Summary:** C1.3.5 removes the backdoor from the image, but it does not remove the trust in its signer. Infected firmware can still be downloaded. Affected versions: `C1.2.7_20250822`, `C1.3.1_20260314`, `C1.3.3_20260611`. Payload removed in: `C1.3.5_20261111`. Platform key: unchanged in all of them.

## Mitigation (what worked for me)

**Best option: update to `C1.3.5_20261111`.** It removes the payload. It also factory-resets the device, so back up first. After the update, I also disabled the two OEM apps with exposed components:

```bash
adb shell pm disable-user --user 0 com.sdt.ota
adb shell pm disable-user --user 0 com.sdt.producttest
```

**If you are still on C1.2.7, C1.3.1 or C1.3.3:** without root you can not delete it from `/system` or `/oem`, but you can hide it for the user:

```bash
adb shell pm uninstall --user 0 com.oemservice
adb shell pm uninstall --user 0 com.oem.qa
adb shell pm uninstall --user 0 com.sdt.backupappmanger
adb shell pm disable-user --user 0 com.sdt.ota
```

In my tests this survived several reboots **and the full C1.3.1 → C1.3.3 OTA**. A factory reset on those builds will very probably bring it back, because the payload is in `/oem`. (I previously also disabled `com.sdt.upgrade`. That is not needed: it turned out to be benign.)

Also block on the router or DNS: `*.anyevonline.com`, `*.szbboys.com`, `bestipip.com`, `intermedia-arts.tv`, `yokatvonline.com`, outgoing TCP 8000, 3885–3887, 16000, 1883, 8883, and the IPs published by Google (`219.134.163.210`, `219.134.163.211`, `38.145.197.135`, `23.94.49.57`).

## Vendor contact

- **2026-11-04:** after a short customer inquiry through their web store, MECOOL sent a standard statement about the XCPE incident from `service@mecool.com` (after-sales technical support). It says MECOOL is an independent brand and the devices are made by Videostrong, that Videostrong appealed to Google, and that after an internal assessment "MECOOL existing products can continue to be used normally and safely". No mention of a firmware fix.
- **2026-11-10:** formal technical report with this advisory sent to MECOOL, with three direct questions.
- **2026-11-11:** MECOOL replied with the same standard statement as on 4 Nov and answered none of the questions.

## Timeline

| Date | Event |
|---|---|
| 2023-12-31 | XCPE 6.5.0 first seen on VirusTotal |
| 2024-07-25 | 6.5.0 installed on my device B (`lastUpdateTime`) |
| 2026-11-02 | Google Threat Intelligence publishes the XCPE takedown |
| 2026-11-04 | MECOOL standard statement to customers ("normally and safely") |
| 2026-11-05/06 | My analysis of both devices, cleanup, repo published |
| 2026-11-06/08 | Device A receives OTA C1.3.3 with the same XCPE |
| 2026-11-10 | Before/after comparison, this advisory; CVE requested from MITRE; formal report to MECOOL |
| 2026-11-11 | MECOOL repeats its standard statement; report to ANCI (Chile) |
| 2026-11-13 | `C1.3.5_20261111` (payload removed, same platform key) published on Google's OTA servers |
| 2026-11-21 | Device A updated to C1.3.5 (automatic wipe) |
| 2026-11-22/23 | Verified: C1.3.5 still uses the same platform key; C1.2.7 package still downloadable |

## References

- Google Threat Intelligence, *Shutting the Door on the XCPE Botnet* (2026-11-02): https://cloud.google.com/blog/topics/threat-intelligence/shutting-the-door-on-the-xcpe-botnet/
- BleepingComputer: https://www.bleepingcomputer.com/news/security/google-disrupts-million-device-xcpe-android-malware-botnet-bans-tv-box-maker/
- My full notes (Spanish): [README.md](README.md)

---

If something here is wrong, please open an issue. I prefer to correct it than to leave a mistake.
