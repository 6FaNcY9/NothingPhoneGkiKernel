# Task: diagnose and propose a fix for Google account sign-in failure on my rooted Nothing Phone (2)

You are working on my Linux box (bandit-lab). The phone is connected over USB with ADB enabled and root access granted to adb shell via the root manager. Work read-only first. Do not change anything on the phone until you have presented a diagnosis and I have approved the fix plan.

## Device and setup

- Nothing Phone (2), codename `Pong`, Snapdragon 8+ Gen 1, stock Nothing OS (Android 15/16 based).
- Custom GKI kernel with KernelSU + SUSFS, root manager is **ReSukiSU** (KernelSU fork). The kernel has been reflashed several times recently.
- "Umount modules by default" is **disabled** globally in the manager because the MFGA font module requires it.
- Installed modules (enabled unless noted):
  - Hybrid Mount Lite 4.2.0 (module mounting, Ext4 mode, Magic:3 Overlay:0)
  - KSU Compatibility Module v1 (Androidacy placeholder)
  - AshReXcue BootLoop Protector 10.0
  - Busybox for Android NDK 1.36.1
  - Frosty Battery Saver 4.5 (GMS service freezer, doze, log killer, kernel tweaks)
  - HMA-OSS Zygisk oss-173 (Hide My Applist backend)
  - MFGA 17.0.1 (system-wide font override)
  - Kali NetHunter 1.4.0 (**disabled**)
  - Play Integrity Fix [INJECT] v4.9-inject-s (KOWX712 fork)
  - ReZygisk v1.0.0
  - SUSFS-FOR-KERNELSU v2.3.0-R28 (susfs4ksu)
  - Tricky Store v1.4.1 (keystore attestation spoof)
  - Google Photos Unlimited Backup v5 (Pixel XL spoof for Photos only)
  - Yurikey Manager v3.0.6 (keybox manager for Tricky Store, action button force-stops GMS)
  - Vector v2.2 Zygisk Xposed framework (**disabled**)

## Symptom

Adding my Google account in Settings fails with:
"Something went wrong. There was a problem communicating with Google servers. Try again later."

History: after several kernel reflashes I ran the Yurikey action (which force-stops Play Services), then I removed the account, cleared cache/storage of Play Services, Google Services Framework and Play Store, and now sign-in fails every time. Network is fine, other apps work.

## Hypotheses to confirm or rule out (in this order)

1. Frosty has frozen GMS components needed for check-in and auth (Cloud / Background categories, "Kill Google Tracking" disables Phenotype). Frosty's own README says the Cloud category breaks Google Sign-in.
2. The device is no longer Play Protect certified: after clearing GSF data the GSF ID changed and Device Integrity now fails, so Google refuses to add an account. Global umount off means GMS can see module mounts; PIF / Tricky Store / Yurikey state may be inconsistent after the reflashes.
3. Private DNS / ad-block DNS or wrong date-time blocking check-in endpoints.
4. HMA-OSS or SUSFS hiding something from GMS in a way that breaks it (less likely).
5. Hybrid Mount Lite or the module stack leaving a broken mount for GMS.

## Diagnostics to run (all via `adb shell`, use `su -c` where root is required)

Collect and keep the raw output in a `diag/` folder on the box.

- Identity: `getprop ro.build.fingerprint`, `getprop ro.product.name`, `getprop ro.boot.verifiedbootstate`, `getprop ro.boot.vbmeta.device_state`, `uname -a`, `getprop ro.kernel.version`.
- Root manager: `su -c "ksud debug version"` (or the ReSukiSU equivalent), `su -c ls /data/adb/modules`, and for each module cat `module.prop` and check for `disable`, `remove`, `skip_mount` files. Dump `/data/adb/ksu/.allowlist` or the profile store if readable to see which apps have umount enabled.
- Frosty state: read its config in `/data/adb/modules/Frosty/` and `/data/adb/frosty*` (find the actual path). Then list disabled GMS components:
  `su -c "pm dump com.google.android.gms | grep -A2000 'disabledComponents' | head -400"` and
  `su -c "pm list packages -d"`. Also `su -c "dumpsys deviceidle whitelist"` and `settings get global device_idle_constants`.
- GMS health: `pm path com.google.android.gms`, `dumpsys package com.google.android.gms | grep -E 'versionName|enabled|stopped|suspended'`, same for `com.google.android.gsf` and `com.android.vending`. `su -c "appops get com.google.android.gms"`.
- Certification: `su -c "sqlite3 /data/data/com.google.android.gsf/databases/gservices.db \"select * from main where name='android_id'\""` (if sqlite3 exists, else use `content query --uri content://com.google.android.gsf.gservices --where "name='android_id'"`). Check `dumpsys package com.google.android.gms | grep -i certif` and look for DroidGuard results in logcat.
- Network/time: `settings get global private_dns_mode`, `settings get global private_dns_specifier`, `settings get global auto_time`, `date`, `su -c "cat /data/adb/modules/*/system/etc/hosts | wc -l"` to spot a hosts-based ad-blocker, `su -c "iptables -L -n -v | head -50"`.
- Tricky Store / PIF / Yurikey: cat `/data/adb/tricky_store/target.txt`, check `/data/adb/tricky_store/keybox.xml` exists (do not print its contents, only size and validity), cat `/data/adb/pif.json` or the PIF module's config, read Yurikey's last log in its module dir.
- Mounts as seen by GMS: `su -c "cat /proc/$(pidof com.google.android.gms | cut -d' ' -f1)/mountinfo | grep -E 'adb|overlay|hybrid|KSU'"` versus `su -c "cat /proc/1/mountinfo | grep -cE 'adb|overlay'"`.
- Reproduction with logs: `adb logcat -c`, I then try to add the account on the phone, you capture `adb logcat -d > diag/signin.log` and grep for `GLSUser|GLSActivity|checkin|Checkin|DeviceIntegrity|DroidGuard|GoogleAuth|AuthZen|NetworkError|StatusCode|SSL|CERT`.

## Deliverable

1. A short root-cause statement backed by the specific lines of output that prove it.
2. A ranked fix plan, smallest change first, each step stating what it changes, how to verify it worked, and how to roll it back. Do not include "reflash the kernel" or "factory reset" unless the evidence leaves nothing else.
3. A list of changes to the module stack that reduce overlap (for example which of Tricky Store / Yurikey / PIF / Frosty settings are redundant or conflicting), kept separate from the fix for this bug.
4. Save the report as `diag/REPORT.md` and show it to me before touching the phone.

## Rules

- Never run `pm clear`, `pm disable`, `rm`, module install/remove, or reboot without my explicit OK for that specific command.
- Do not print keybox contents, account tokens, GSF android_id beyond the last 4 characters, or anything from `/data/data/*/shared_prefs` that looks like a credential.
- If adb root is unavailable, say so and continue with what non-root `adb shell` can collect.
