# Safety Considerations
[Back to Home](../00_Home.md)

## Safety Statement

NeuroFit is an experimental wellness platform. It is not approved as a medical device and must not be used for clinical diagnosis or treatment decisions.

## Electrical and Wearability Safety

1. Use medically appropriate isolation and front-end protection for biopotential sensing.
2. Ensure skin-contact materials are hypoallergenic and cleaned between sessions.
3. Prevent overheating and excessive mechanical pressure in headband design.

## Biosignal Interpretation Warnings

- EEG band percentages are coarse wellness indicators, not diagnostic EEG.
- Motion artifacts may appear as abnormal patterns.
- SpO2 and HR alerts are only as reliable as sensor contact quality and calibration.

## Emergency Workflow Limitations

| Limitation | Risk |
|---|---|
| Dependence on connectivity (BLE/web/API) | Delayed or missed alert dispatch |
| Threshold-only logic | False positives and false negatives |
| Browser geolocation permissions | Missing location context in alerts |

## Recommended Safe-Use Policy

1. Keep explicit user-facing disclaimers in product UI.
2. Require informed consent for data collection and alert contacts.
3. Implement local fallback alerts (haptic/audio) if cloud paths fail.
4. Maintain incident logs for post-event review.

See [Testing-and-Validation]() and [FAQ](../04_Community_and_Gevernance/20_FAQ.md).
