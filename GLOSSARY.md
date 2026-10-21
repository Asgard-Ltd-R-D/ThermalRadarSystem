# ThermalRadarSystem

Adds thermal-camera evidence to the Radar System's Suspect Regions to decide whether a Person carries an Anomaly, and raises Alerts.

## Language

**Radar System**:
The PulseR DeviceDetection stack on pulse72: three 4D radars plus an RGB camera that together produce Suspect Regions.
_Avoid_: "the radar" when the whole stack is meant

**Person**:
A human the Radar System is tracking, known by a temporary track ID that is not a real identity.
_Avoid_: target, subject

**Anomaly**:
A concealed object on a Person. The system reports Anomaly or no Anomaly, never the object's type.
_Avoid_: weapon, threat

**Suspect Region**:
A region on a Person where the Radar System's evidence points to a possible Anomaly, tracked across frames.
_Avoid_: weapon detection, ROI, radar detection

**Thermal Verification**:
The thermal model's judgement of whether a given Suspect Region shows an Anomaly in the thermal image.
_Avoid_: thermal detection (the thermal side never detects on its own)

**Verification Zone**:
The range band where Thermal Verification can see a Suspect Region: about 1.5–3 m from the system, near the centre line of the path.
_Avoid_: thermal coverage, thermal FOV

**Verdict**:
Fusion's conclusion about one Suspect Region: Confirmed (Thermal Verification agrees), Cleared (it disagrees) or Unverified (it never got enough usable looks).
_Avoid_: decision, result, rejected, unconfirmed

**Alert**:
A notification to a human operator that a Person may carry an Anomaly; the system takes no other action.
_Avoid_: alarm (the Radar System's own word for a long-lived Suspect Region)
