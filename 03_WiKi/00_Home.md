# NeuroFit Wiki

Welcome to the NeuroFit technical wiki. This documentation describes the full wearable stack: embedded firmware, biosignal pipeline, BLE transport, web application, safety constraints, and contributor workflows.

NeuroFit is a wearable mind and health tracker that combines:

- EEG-style biosignal capture (BioAmp EXG input)
- Heart-rate and SpO2 monitoring (planned MAX30102 integration)
- On-device fall/emergency detection (IMU-assisted)
- BLE streaming to a web dashboard
- Alert escalation via email/SMS

## Quick Navigation

### Start Here

- [Project Overview](01_Start_Here/01_Project-Overview.md)
- [System Architecture](01_Start_Here/02_System-Architecture.md)
- [Roadmap](01_Start_Here/03_Roadmap.md)

### Architecture and Engineering

- [Hardware Architecture](02_Architecture_and_Engineering/04_Hardware-Architecture.md)
- [Software Architecture](02_Architecture_and_Engineering/05_Software-Architecture.md)
- [Sensor Modules](02_Architecture_and_Engineering/06_Sensor-Modules.md)
- [Circuit Diagram](02_Architecture_and_Engineering/07_Circuit-Diagram.md)
- [Pinout](02_Architecture_and_Engineering/08_Pinout.md)
- [Data Flow](02_Architecture_and_Engineering/09_Data-Flow.md)
- [Signal Processing](02_Architecture_and_Engineering/10_Signal-Processing.md)
- [BLE Communication](02_Architecture_and_Engineering/11_BLE-Communication.md)
- [Power Management](02_Architecture_and_Engineering/12_Power-Management.md)

### Setup and Operations

- [Firmware Installation](03_Setup_and_Operations/13_Firmware-Installation.md)
- [Build Instructions](03_Setup_and_Operations/14_Build-Instructions.md)
- [Safety Considerations](03_Setup_and_Operations/15_Safety-Considerations.md)
- [Testing and Validation](03_Setup_and_Operations/16_Testing-and-Validation.md)
- [Troubleshooting](03_Setup_and_Operations/17_Troubleshooting.md)

### Community and Governance

- [Contributing](04_Community_and_Gevernance/18_Contributing.md)
- [License](04_Community_and_Gevernance/19_License.md)
- [FAQ](04_Community_and_Gevernance/20_FAQ.md)

## Repository-at-a-Glance

| Path | Purpose |
|---|---|
| `Hardware/01-first-firmware/NeuroFit/NeuroFit.ino` | Firmware: BLE, EEG processing, emergency/fall logic |
| `Web (flask)/app.py` | Flask backend, auth, APIs, emergency alert routing |
| `Web (flask)/templates/` | Dashboard, health data, meditation UI pages |
| `Web (flask)/static/js/` | Web Bluetooth, dashboard logic, location, meditation controls |
| `Web (flask)/requirements.txt` | Python package dependencies |
| `Web (flask)/Dockerfile` | Containerized runtime configuration |
| `Docs/` | Additional project documentation assets |

## Current Maturity Summary

- Firmware has real FFT-based EEG band analysis and BLE characteristic writes.
- Frontend and firmware BLE profiles are partially misaligned and require harmonization.
- Some sensor streams are still simulated in software placeholders.
- Security hardening (credentials, password hashing, secrets management) is required before production use.

For implementation gaps and planned upgrades, see [Roadmap](01_Start_Here/03_Roadmap.md).
