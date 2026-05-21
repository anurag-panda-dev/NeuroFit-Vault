# Sensor Modules
[Back to Home](../00_Home.md)

## Module Inventory

| Sensor | Signal Type | Interface | Primary Metrics | Notes |
|---|---|---|---|---|
| BioAmp EXG Pill | Analog biopotential | ADC (A1) | Bandpower-derived EEG state | High sensitivity to electrode quality |
| MAX30102 | Optical PPG | I2C | Heart rate, SpO2 | Firmware currently uses simulated values in update path |
| LSM6DS3 | 6-axis IMU | I2C | Acceleration, orientation change | Used for fall detection workflow |

## EEG / EXG Capture Notes

- Sampling target: 250 Hz
- FFT window size: 128 samples
- Bands extracted: Delta, Theta, Alpha, Beta, Gamma
- Output format: percentage contribution per band + dominant band index

## Calibration Guidance

| Sensor | Calibration Action | Frequency |
|---|---|---|
| EXG | Baseline noise capture, electrode impedance check | Before each session |
| MAX30102 | Finger/skin contact and ambient light shielding check | Per wear session |
| IMU | Static bias check and orientation sanity test | Weekly or after mechanical changes |

## Artifact Sources

1. Motion artifacts from head movement and cable microphonics.
2. Contact quality drift due to sweat, hair, or electrode displacement.
3. Power rail noise coupling into analog front-end.

See [Signal-Processing]() for filtering and feature extraction strategy.
