---
date: "2025-12-12T08:45:11+07:00"
draft: false
summary: "Change car emulator make & model"
title: "Change car emulator make & model"
categories:
  - Code
tags:
  - car
  - emulator
---

## Prerequisites

- Android SDK installed with `emulator` and `adb` in PATH
- An **non-google-service** Android Automotive AVD created (e.g., Automotive_1408p_landscape)

## Steps

### Boot emulator with writable system

1. List available AVDs:

```bash
emulator -list-avds
```

2. Start the emulator with a writable system image:

```bash
emulator -avd Automotive_1408p_landscape -writable-system
```

3. Get root and remount:

```bash
adb root
adb remount
```

### Modify make & model

1. Pull the properties file:

```bash
adb pull /vendor/etc/automotive/vhalconfig/DefaultProperties.json .
```

2. Edit DefaultProperties.json locally to change make/model (use your editor).

```json
{
    "property": "VehicleProperty::INFO_MAKE",
    "defaultValue": {
        "stringValue": "Toyota"
    }
},
{
    "property": "VehicleProperty::INFO_MODEL",
    "defaultValue": {
        "stringValue": "Camry"
    }
},
```

3. Push the modified file back to the device:

```bash
adb push DefaultProperties.json /vendor/etc/automotive/vhalconfig/DefaultProperties.json
```

4. Reboot the emulator to apply changes:

```bash
adb reboot
```
