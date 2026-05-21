# Contributing
[Back to Home](../00_Home.md)

## Contribution Philosophy

NeuroFit welcomes contributions across embedded systems, web engineering, biosignal processing, QA, and documentation.

## Development Workflow

1. Fork and create a focused feature branch.
2. Keep commits atomic and message intent clearly.
3. Update wiki/docs for architecture or behavior changes.
4. Open a pull request with test evidence and risk notes.

## Pull Request Checklist

| Item | Required |
|---|---|
| Clear problem statement and scope | Yes |
| Linked issue or rationale | Yes |
| Test or validation evidence | Yes |
| Backward compatibility impact | Yes |
| Documentation updates | Yes |

## Coding Standards

- Firmware: deterministic timing, minimal blocking delays in hot loops.
- Backend: no hardcoded secrets, explicit error handling.
- Frontend: resilient BLE handling and graceful degradation.
- Security: never commit credentials, tokens, or private endpoints.

## Priority Contribution Areas

1. Sensor driver completeness and calibration utilities.
2. BLE profile stabilization and parser hardening.
3. Automated tests for emergency and signal-processing flows.
4. Clinical-quality safety and validation documentation.

See [Build-Instructions]() and [Roadmap](../01_Start_Here/03_Roadmap.md).
