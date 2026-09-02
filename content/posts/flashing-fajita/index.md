---
categories:
  - Code
date: "2025-12-29T10:59:38+07:00"
draft: false
summary: Flashing LineageOS on OnePlus 6T (fajita) using ADB Sideload
tags:
  - adb
  - android
title: Flashing LineageOS on OnePlus 6T (fajita)
---

## Prerequisites

- Download LineageOS from https://download.lineageos.org/devices/fajita/builds
- Download GApps (arm64) from https://wiki.lineageos.org/gapps/
- Download Magisk from https://github.com/topjohnwu/Magisk/releases (rename the APK to .zip)

## Steps

```bash
adb -d reboot sideload
adb -d sideload lineage-20.0-20240624-nightly-fajita-signed.zip
# Recovery → Advanced → Enable ADB (do not reboot yet)
adb -d reboot sideload
adb -d sideload MindTheGapps-15.0.0-arm64-20250812_214357.zip
adb -d reboot sideload
adb -d sideload Magisk-v26.4.zip
adb -d reboot
```

## Troubleshooting

**Device Offline Error During Sideload**

- Ensure the device is connected via USB cable
- Verify USB debugging is enabled
