# Kindle/Fire tablet debloat via adb

Direct `adb` equivalent of what Fire Toolbox does, for an Amazon Fire HD 8
(5th gen, 2015) running Fire OS 5 / Android 5.1. No root required —
`pm disable-user` is fully reversible. A factory reset restores everything.

## Prerequisites

- A **data-capable** micro-USB cable (charge-only cables are the #1 failure).
- ADB installed (`brew install android-platform-tools` or Android SDK).
- On the tablet: Settings → Device Options → tap **Serial Number** 7 times →
  Developer Options → **Enable ADB**.

## 1. Verify the connection

```bash
adb devices    # device id should show "device", not "unauthorized"
```

If `unauthorized`, accept the "Allow USB debugging?" prompt on the tablet
(tick "Always allow from this computer"). Confirm the model:

```bash
adb shell getprop ro.product.model
adb shell getprop ro.build.version.release   # expect 5.1.x on the 2015 HD 8
```

## 2. Snapshot RAM before debloating

```bash
adb shell cat /proc/meminfo | head -3
```

## 3. Debloat

Package names vary slightly by Fire OS build — check what's on yours first:

```bash
adb shell pm list packages | grep amazon | sort
```

### Group A — safe bloat

```bash
for p in \
  com.amazon.music \
  com.amazon.mp3 \
  com.amazon.photos \
  com.amazon.photos3 \
  com.amazon.ags.app \
  com.amazon.goodreads \
  com.amazon.drive \
  com.amazon.geo.client \
  com.amazon.geo.maps \
  com.amazon.device.backup \
  com.amazon.kindle.devicecontrols \
  com.amazon.kindle.socialshare \
  com.amazon.unifiedsharesheet \
  com.amazon.sharingservice.android \
  com.amazon.kindle.kso \
; do adb shell pm disable-user --user 0 $p 2>/dev/null && echo "disabled $p" || echo "not found  $p"; done
```

`com.amazon.kindle.kso` is the **lockscreen ads** service (Kindle Special
Offers). The lockscreen may go plain/black afterward; harmless.

### Group B — telemetry / background chatter

```bash
for p in \
  com.amazon.client.metrics \
  com.amazon.sync \
  com.amazon.kindle.personalyst \
  com.amazon.securityhub \
; do adb shell pm disable-user --user 0 $p 2>/dev/null && echo "disabled $p" || echo "not found  $p"; done
```

`com.amazon.securityhub` is the "your device is at risk" nag service.

### Group C — optional, only if you won't use it

- `com.amazon.venezia` — the Amazon Appstore. No Amazon app updates after
  this; fine only if going all-in on Google Play.
- `com.amazon.cloud9` — the Silk browser. Keep it if no other browser is
  installed.
- `com.amazon.avod` — Prime Video, if unused.

### Do NOT disable

- `com.amazon.kindle` — on Fire OS 5 this **is** the home screen launcher;
  disabling it bricks the UI.
- `com.amazon.dcp`, `com.amazon.platform` — core services.
- `com.amazon.device.settings`, `com.amazon.kindle.setup` — settings and
  first-run setup.

## 4. Kill animations

Same as Developer Options → animation scales, but from the shell:

```bash
adb shell settings put global window_animation_scale 0.5
adb shell settings put global transition_animation_scale 0.5
adb shell settings put global animator_duration_scale 0.5
```

## 5. Verify and reboot

```bash
adb shell cat /proc/meminfo | head -3     # compare to step 2
adb shell pm list packages -d             # everything currently disabled
adb reboot
```

## Undo

```bash
adb shell pm enable com.amazon.music      # one package per line
adb shell pm list packages -d             # see what's disabled
```

## Not covered by adb alone

- **Google Play install** — a specific sequence of pushed APKs
  (`adb push` + `pm install`) targeting Android 5.1; Google ended official
  support for Android 5 in 2024, so do this while it still works.
- **Launcher replacement** (Nova via LauncherHijack) — requires sideloading
  an APK.
