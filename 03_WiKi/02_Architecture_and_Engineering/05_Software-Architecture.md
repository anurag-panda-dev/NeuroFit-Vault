# Software Architecture
[Back to Home](../00_Home.md)

## Layers

```mermaid
flowchart TB
  FW[Firmware (Arduino/C++)]
  WEB[Web Client (HTML/CSS/JS)]
  API[Flask API]
  DATA[(MySQL)]
  ALERT[Alert Integrations]

  FW --> WEB
  WEB --> API
  API --> DATA
  API --> ALERT
```

## Backend (Flask)

### Key Modules

| Module | Purpose |
|---|---|
| Auth routes (`/login`, `/signup`, `/logout`) | Session-based user access control |
| Dashboard routes (`/dashboard`, `/health-data`, `/meditation`) | Render application views |
| API routes (`/api/health-data`, `/api/meditation/status`, `/api/emergency-alert`) | Data and event endpoints |
| SQLAlchemy models (`User`, `HealthData`) | User and telemetry persistence |

## Frontend

| Script | Responsibility |
|---|---|
| `bluetooth.js` | BLE pairing and characteristic notification setup |
| `dashboard.js` | Dashboard metric/chart updates |
| `location.js` | Geolocation collection for emergency payloads |
| `meditation.js` | Timer, audio controls, real-time session charting |

## Firmware Modules

| Function Group | Key Functions |
|---|---|
| BLE setup/publish | `setupBLE()`, characteristic `.write(...)` |
| EEG DSP | `processEEG()`, `sendEEGData(...)` |
| Vitals + battery | `updateHRSpO2()`, `checkBattery()` |
| Emergency detection | `checkEmergencies()`, `triggerEmergencyAlert(...)` |
| Fall logic | `setupIMU()`, `isFallingDetected()` |

## Architectural Gaps to Address

1. Align firmware BLE UUID map with browser BLE service/characteristic discovery.
2. Replace hardcoded credentials with environment-variable configuration.
3. Add schema/versioning for telemetry payload contracts.

See [Build-Instructions]() and [Contributing](../04_Community_and_Gevernance/18_Contributing.md).
