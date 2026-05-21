# Project Overview
[Back to Home](../00_Home.md)

## Mission

NeuroFit targets continuous personal wellness monitoring by combining physiological sensing, edge inference primitives, and user-facing guidance workflows in a wearable form factor.

## High-Level Objectives

1. Capture biosignals and vital signs from a headband form factor.
2. Stream real-time telemetry over BLE to a web experience.
3. Derive interpretable signal features (EEG band percentages).
4. Detect emergencies and trigger escalation workflows.
5. Support longitudinal trend analysis and mindfulness sessions.

## Scope in Current Repository

| Domain | Implemented | In Progress / Planned |
|---|---|---|
| Embedded firmware | BLE service + characteristics, EEG FFT, battery estimation, fall logic | Sensor driver completion for MAX30102, robust sampling scheduler |
| Web app | Flask routes, auth flow, dashboard templates, emergency API | Production auth hardening, richer analytics API |
| Data persistence | SQLAlchemy models (`User`, `HealthData`) | Historical query implementation and indexing |
| Alerts | Email (Apps Script) + SMS (TextBee) dispatch | Retry policy, observability, secure key storage |
| Meditation | Session UI + audio + simple charting | Biofeedback-driven adaptive session engine |

## Biomedical Context

NeuroFit currently implements *wellness-grade* telemetry logic. It is **not** a certified medical device and must not be used for diagnosis, treatment, or emergency triage decisions without clinical validation.

See [Safety Considerations](../03_Setup_and_Operations/15_Safety-Considerations.md) for detailed guidance.

## Related Pages

- [System Architecture](02_System-Architecture.md)
- [Sensor Modules](../02_Architecture_and_Engineering/06_Sensor-Modules.md)
- [Signal Processing](../02_Architecture_and_Engineering/10_Signal-Processing.md)
- [Testing and Validation](../03_Setup_and_Operations/16_Testing-and-Validation.md)
