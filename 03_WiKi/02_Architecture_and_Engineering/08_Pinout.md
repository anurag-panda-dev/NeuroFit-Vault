# Pinout
[Back to Home](../00_Home.md)

## MCU Pin Mapping

| MCU Pin | Direction | Signal | Connected Module | Firmware Reference |
|---|---|---|---|---|
| A1 | Input (ADC) | EEG_INPUT_PIN | BioAmp EXG analog output | `#define EEG_INPUT_PIN A1` |
| A0 | Input (ADC) | BATTERY_PIN | Battery voltage divider | `#define BATTERY_PIN A0` |
| D2 | Output (GPIO) | VIBRATION_PIN | Vibration motor driver | `#define VIBRATION_PIN D2` |
| I2C SDA/SCL | Bi-directional | Sensor bus | LSM6DS3 IMU, MAX30102 target | `setupIMU()` and future MAX30102 init |

## BLE Logical Endpoints (Firmware)

| Name | UUID | Data Length | Purpose |
|---|---|---|---|
| Health Service | `1810` | N/A | Container service for health telemetry |
| Heart Rate Char | `2A37` | 2 bytes | Heart rate notifications |
| SpO2 Char | `2A5F` | 2 bytes | Oxygen saturation notifications |
| EEG Char | `2A38` | 20 bytes | EEG feature packet |
| Battery Char | `2A19` | 1 byte | Battery percentage |

See [BLE-Communication]() for payload and parser details.
