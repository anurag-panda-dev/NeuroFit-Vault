# FAQ
[Back to Home](../00_Home.md)

## Is NeuroFit a medical device?

No. NeuroFit is an experimental wellness/research platform and is not certified for diagnosis or treatment.

## Which sensors are currently active in firmware?

EEG analog processing and IMU fall logic are active. HR/SpO2 logic exists but includes simulated data paths pending full MAX30102 driver integration.

## Why is BLE not always showing all values?

Current frontend BLE service/characteristic assumptions are not fully aligned with firmware UUID definitions. See [BLE-Communication]().

## Where are emergency alerts sent?

The backend can dispatch:

- Email via Google Apps Script endpoint
- SMS via TextBee API

## How is mood estimated?

The present codebase provides a placeholder mood-score path in the web layer. A validated mood model is planned in future roadmap phases.

## What are the biggest technical risks right now?

1. BLE contract mismatch across firmware and frontend.
2. Hardcoded secrets in backend code.
3. Incomplete sensor integration and limited validation coverage.

## How can I help?

Start with [Contributing](18_Contributing.md), then pick items from [Roadmap](../01_Start_Here/03_Roadmap.md) and submit focused pull requests with validation evidence.
