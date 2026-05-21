# Firmware Installation
[Back to Home](../00_Home.md)

## Target Platform

- Board family: Seeed XIAO nRF52840 (nRF52)
- Language/runtime: Arduino framework (C/C++)
- Main sketch: `Hardware/01-first-firmware/NeuroFit/NeuroFit.ino`

## Prerequisites

1. Arduino IDE 2.x
2. Adafruit nRF52 board support package
3. USB data cable
4. Installed libraries:
   - `Adafruit Bluefruit nRF52`
   - `movingAvg`
   - `arduinoFFT`
   - `Adafruit LSM6DS3`

## Flash Procedure

1. Open Arduino IDE and load `NeuroFit.ino`.
2. Select board: XIAO nRF52840.
3. Select correct COM port.
4. Verify/compile the sketch.
5. Upload and monitor serial output at `115200` baud.

## Post-Flash Validation Checklist

| Check | Expected Result |
|---|---|
| BLE advertising | Device name visible as `MindHealth_Tracker` |
| Serial startup | `BLE Ready` message appears |
| EEG loop | Periodic packet writes occur after sample buffer fill |
| Battery warnings | Low battery triggers double buzz pattern |

## Common Bring-Up Issues

- Missing libraries: install exact dependencies in Arduino Library Manager.
- Upload failures: press reset/bootloader sequence and retry.
- No BLE discoverability: verify board selection and successful `setupBLE()` execution.

See [Troubleshooting](17_Troubleshooting.md) for an expanded matrix.
