# Power Management
[Back to Home](../00_Home.md)

## Implemented Mechanisms

| Mechanism | Description |
|---|---|
| Battery ADC read | Firmware samples battery channel (`A0`) |
| Voltage-to-percent map | Linear map using 3.0 V to 3.7 V bounds |
| Low battery feedback | Haptic double-buzz when battery < 20% |

## Consumption Contributors

1. BLE advertising/connected notification duty cycle
2. High-rate sampling and FFT compute load
3. IMU continuous polling
4. Vibration motor actuation bursts

## Optimization Opportunities

| Optimization | Expected Benefit |
|---|---|
| BLE connection interval tuning | Lower radio duty cycle |
| Sensor scheduling with sleep windows | Reduced average current |
| Conditional IMU sampling | Less processing overhead |
| Adaptive haptic pulse duration | Lower peak energy draw |

## Measurement Recommendations

- Measure current in states: idle, BLE connected, active FFT, alert vibration.
- Build battery-life model with realistic duty-cycle assumptions.
- Track thermal behavior during prolonged sessions.

See [Testing-and-Validation]().
