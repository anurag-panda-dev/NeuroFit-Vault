# Circuit Diagram
[Back to Home](../00_Home.md)

## Logical Wiring Diagram

```mermaid
flowchart LR
  MCU[nRF52840]
  EXG[BioAmp EXG]
  MAX[MAX30102]
  IMU[LSM6DS3]
  BATSENSE[Battery Divider]
  MOTOR[Vibration Motor + Driver]

  EXG -->|A1 Analog Out| MCU
  BATSENSE -->|A0 ADC| MCU
  MAX <-->|I2C SDA/SCL| MCU
  IMU <-->|I2C SDA/SCL| MCU
  MCU -->|D2 GPIO| MOTOR
```

## Wiring Table (Current/Target)

| Signal | MCU Pin | Connected Module | Status |
|---|---|---|---|
| EEG Analog | A1 | BioAmp EXG output | Implemented |
| Battery ADC | A0 | Battery divider output | Implemented |
| Haptic Control | D2 | Vibration motor driver input | Implemented |
| I2C SDA/SCL | Board default I2C pins | MAX30102 + LSM6DS3 | LSM6DS3 in code; MAX30102 integration pending |

## Electrical Recommendations

1. Add RC low-pass and proper shielding for EXG analog line.
2. Use dedicated transistor/MOSFET + flyback strategy for motor drive stability.
3. Verify battery divider resistor values to keep ADC input within safe range.

For exact pin references and firmware mapping, see [Pinout](08_Pinout.md).
