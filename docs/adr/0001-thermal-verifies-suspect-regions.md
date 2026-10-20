# Thermal verifies Suspect Regions; it is not an independent detector

The first brief had the radar find people and the thermal camera find anomalies. The customer's notes (`docs/meyuhadim/thermal_weapon_customer_meeting_questions_he_updated_v2.md`) and the Radar System's code (`docs/research/pulse72-radar-system.md`) both show the radar side already outputs Suspect Regions on tracked Persons. So the thermal model only answers "does this Suspect Region show an Anomaly?". Fusion becomes "project the region into the thermal image and check it" instead of a general multi-sensor tracker, and the thermal model can be trained on region crops.

## Consequences

- Thermal can never raise an Alert for a region the Radar System missed.
- Independent thermal detection with voting was rejected for v1: it needs its own person tracking and cross-sensor association.
