# System Architecture
[Back to Home](../00_Home.md)

## Architecture Summary

NeuroFit is organized as a three-layer system:

1. Wearable edge node (microcontroller + sensors + actuator)
2. Wireless transport (BLE GATT notifications)
3. Web platform (Flask backend + browser clients + alert gateways)

## End-to-End Block Diagram

```mermaid
flowchart LR
  subgraph Wearable[Wearable Device]
    MCU[XIAO nRF52840 Firmware]
    EEG[BioAmp EXG Input]
    IMU[LSM6DS3 IMU]
    PPG[MAX30102 HR/SpO2]
    VIB[Vibration Motor]
    EEG --> MCU
    IMU --> MCU
    PPG --> MCU
    MCU --> VIB
  end

  MCU -->|BLE Notifications| Browser[Web Dashboard]
  Browser -->|REST API| Flask[Flask Backend]
  Flask --> DB[(MySQL via SQLAlchemy)]
  Flask --> Email[Google Apps Script Mail]
  Flask --> SMS[TextBee SMS Gateway]
```

## Functional Responsibilities

| Layer | Responsibilities |
|---|---|
| Firmware | Sampling, FFT feature extraction, threshold checks, BLE publication, haptic alerts |
| Browser | Device pairing, live rendering, meditation UX, location capture |
| Backend | Authentication, data ingestion APIs, persistence, emergency dispatch |

## Interface Boundaries

- BLE profile and payload schema between firmware and browser
- REST contracts between browser and Flask (`/api/*`)
- Alert payload contracts for email/SMS providers

## Known Integration Risks

| Risk | Impact | Mitigation |
|---|---|---|
| BLE UUID mismatch (firmware vs web code) | No/partial telemetry | Standardize UUID map and parser contract |
| Simulated sensor values in code paths | Invalid analytics | Replace placeholders with calibrated drivers |
| Hardcoded secrets | Security exposure | Move to environment variables + secret vault |

See [BLE Communication](../02_Architecture_and_Engineering/11_BLE-Communication.md) and [Roadmap](03_Roadmap.md).
