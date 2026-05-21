# Data Flow
[Back to Home](../00_Home.md)

## Pipeline Overview

```mermaid
flowchart LR
  A[Sensor Sampling] --> B[On-device Feature Extraction]
  B --> C[Emergency Threshold Checks]
  C --> D[BLE Characteristic Notifications]
  D --> E[Browser Decoding + UI Update]
  E --> F[Flask API Ingestion]
  F --> G[Database Storage]
  F --> H[Email/SMS Emergency Dispatch]
```

## Operational Steps

1. Firmware samples analog EEG and motion data.
2. FFT converts raw EEG window into band percentages.
3. Firmware publishes HR/SpO2/EEG/battery via BLE notifications.
4. Browser dashboard receives characteristic events and updates metrics.
5. Backend API persists selected data and triggers emergency workflows.

## State Flow (Emergency Path)

```mermaid
stateDiagram-v2
  [*] --> Monitoring
  Monitoring --> AlertCandidate: Threshold breach
  AlertCandidate --> ConfirmedAlert: Condition sustained/validated
  ConfirmedAlert --> NotifyUser: Haptic pattern
  ConfirmedAlert --> NotifyContacts: API email/SMS
  NotifyContacts --> Monitoring
```

## Data Integrity Considerations

- Add timestamps at source and at ingestion.
- Include sensor validity flags and confidence metrics.
- Store firmware and parser schema versions alongside records.

See [Testing-and-Validation]() for verification strategy.
