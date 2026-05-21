# Troubleshooting
[Back to Home](../00_Home.md)

## Troubleshooting Matrix

| Symptom | Likely Cause | Diagnostic Steps | Resolution |
|---|---|---|---|
| Device not visible over BLE | Advertising not started or wrong board flash | Check serial log for startup and BLE readiness | Reflash firmware, verify `setupBLE()` completes |
| Connect succeeds but no telemetry | UUID/characteristic mismatch | Compare frontend requested UUIDs with firmware UUIDs | Standardize UUID map and update parser |
| Frequent emergency alerts | Motion artifacts or aggressive thresholds | Review raw signal quality and movement context | Add filtering, tune thresholds, debounce events |
| SpO2/HR values unrealistic | Placeholder simulation path active | Inspect firmware `updateHRSpO2()` logic | Integrate real MAX30102 driver and calibration |
| No SMS sent | API key/device ID invalid or network issue | Check API responses and HTTP errors | Move secrets to env vars and verify account settings |
| Login works but weak security | Plaintext password storage | Inspect auth model and login flow | Add salted hashing and secure session settings |

## Frontend-Specific Issues

1. Browser blocks Bluetooth on insecure origin: use HTTPS or localhost.
2. Geolocation denied: emergency payload lacks location.
3. Missing script file references can break charts and data pages.

## Firmware-Specific Issues

1. Loop timing jitter can affect EEG sampling quality.
2. Blocking delays in alert patterns may reduce responsiveness.
3. IMU initialization should be explicitly invoked at startup.

For test-driven diagnosis, see [Testing-and-Validation]().
