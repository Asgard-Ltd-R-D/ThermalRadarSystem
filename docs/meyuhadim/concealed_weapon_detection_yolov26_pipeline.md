# Concealed Threat Detection Pipeline — YOLOv26

## Goal

Detect whether a person may be carrying a concealed threat such as:

- **Large Knife**
- **Large Gun**
- **Explosive Device**

The first MVP can use a **thermal camera**, but an important limitation is:

> A normal thermal camera does **not truly see through clothing**.

It mainly measures the temperature of the clothing surface.

A concealed object may sometimes create a visible thermal pattern because it changes the temperature distribution of the clothing above it.

For a production system where true concealed-object detection is required, a **millimeter-wave (MMWave) sensor** is a better choice.

---

# 1. Recommended Thermal MVP Pipeline

```text
THERMAL CAMERA
      ↓
Raw Thermal Frame
      ↓
Thermal Preprocessing
      ↓
YOLOv26s
Person Detection
      ↓
ByteTrack
      ↓
Track ID
      ↓
Crop Person
      ↓
YOLOv26 Pose
      ↓
Body Regions / ROIs
      ↓
Chest / Waist / Back / Legs
      ↓
Thermal Normalization
      ↓
YOLOv26m
Concealed Object Detector
      ↓
No Threat / Large Knife / Large Gun / Explosive Device
      ↓
Temporal Verification
      ↓
Same Result Across Multiple Frames?
      ↓
YES
      ↓
ALERT
```

---

# 2. Thermal Camera Input

Prefer obtaining the raw thermal frame rather than only the colored display image.

Example:

```text
Preferred:

Thermal Camera
      ↓
Raw grayscale / radiometric thermal image
```

Instead of:

```text
Thermal Camera
      ↓
Rainbow-colored visualization
```

The neural network should learn from the underlying thermal information rather than from an arbitrary display color palette.

---

# 3. Thermal Preprocessing

A basic preprocessing pipeline can be:

```text
Raw Thermal Frame
      ↓
Bad / Dead Pixel Correction
      ↓
NUC / Sensor Correction
      ↓
Temperature Normalization
      ↓
Contrast Normalization
      ↓
YOLOv26
```

## RGB vs Thermal

Normal RGB:

```text
R
G
B
```

Thermal:

```text
Temperature / Intensity
```

Thermal data is usually effectively a single meaningful channel.

For an easy first experiment with pretrained YOLO weights:

```text
Thermal Channel
      ↓
Duplicate Channel
      ↓
[T, T, T]
      ↓
YOLOv26
```

This allows you to reuse a model initialized for 3-channel input.

Later, you can modify the network input layer to accept a native single-channel thermal image.

---

# 4. Person Detection

Recommended starting model:

```text
YOLOv26s
```

Use it to detect the person before trying to detect concealed objects.

```text
Thermal Frame
      ↓
YOLOv26s
      ↓
Person Bounding Box
```

Then crop the person:

```text
Full Thermal Image
      ↓
Person Detection
      ↓
Person Crop
      ↓
Concealed Object Detection
```

This reduces background false positives.

---

# 5. Tracking with ByteTrack

After YOLOv26 detects the person:

```text
YOLOv26
      ↓
Person Detection
      ↓
ByteTrack
      ↓
Track ID
```

Example:

```text
Person → Track ID 5
```

ByteTrack helps answer:

> Is this the same person that I saw in the previous frame?

The Track ID is temporary.

```text
Track ID 5 ≠ Real Identity
```

It simply lets the system follow the same person over time.

---

# 6. Body Region Detection

A concealed-object detector should not necessarily search every pixel of the entire image.

Recommended model:

```text
YOLOv26 Pose
```

Use the pose landmarks to estimate regions such as:

```text
        HEAD
         🙂
         │
   ┌───────────┐
   │   CHEST   │
   └───────────┘
   ┌───────────┐
   │   WAIST   │
   └───────────┘
      /     \
     /       \
 UPPER LEG  UPPER LEG
```

Useful regions may include:

- Chest
- Abdomen
- Waist
- Back
- Upper legs

Then:

```text
Person Crop
      ↓
YOLOv26 Pose
      ↓
Body ROIs
      ↓
Concealed Object Detector
```

---

# 7. Concealed Object Detection

Recommended model:

```text
YOLOv26m
```

The concealed thermal pattern may be subtle, so this task is harder than basic person detection.

Pipeline:

```text
Body ROI
      ↓
YOLOv26m
      ↓
Bounding Box
+
Class
+
Confidence
```

Example classes:

```text
no_concealed_threat
large_knife
large_gun
explosive_device
```

---

# 8. Start With Simple Classes

Do not initially try to classify exact weapon models such as:

```text
M16
AK-47
M4
Glock 17
etc.
```

The thermal sensor may not contain enough information to reliably distinguish exact models under clothing.

A better first classification scheme is:

```text
0 = no_concealed_threat
1 = large_knife
2 = large_gun
3 = explosive_device
```

Or for the very first MVP:

```text
0 = no_weapon
1 = concealed_threat
```

Once the detector works reliably, expand to:

```text
concealed_threat
      ↓
 ┌──────────┬───────────┬──────────────┐
 │  Large Knife │ Large Gun │ Explosive Device│
 └──────────┴───────────┴──────────────┘
```

---

# 9. P2 Detection Head

For this project, a **P2 head may become useful**.

YOLO uses different feature-map resolutions:

```text
P2
P3
P4
P5
```

A small thermal anomaly may occupy only a small number of pixels.

Example:

```text
Person Crop = 640 × 640

Concealed Knife Signature
≈ 15 × 40 pixels
```

A possible future detector:

```text
YOLOv26
      ↓
Backbone
      ↓
Feature Pyramid
      ↓
┌────┬────┬────┬────┐
│ P2 │ P3 │ P4 │ P5 │
└────┴────┴────┴────┘
 ↓
Small thermal patterns
```

Important:

> P2 is not more important than P3/P4/P5.

It simply gives the detector another higher-resolution perspective that can help with small objects.

Start without modifying the architecture.

Only add P2 if experiments show that the concealed signatures are consistently too small for the standard model.

---

# 10. Temporal Verification

Do not trigger an alarm because of one frame.

Bad approach:

```text
Frame 100
      ↓
Weapon Confidence = 0.63
      ↓
ALERT
```

Instead use multiple frames.

Example:

```text
Track 5

Frame 100 → large_gun = 0.79
Frame 101 → large_gun = 0.83
Frame 102 → large_gun = 0.77
Frame 103 → large_gun = 0.84
Frame 104 → large_gun = 0.81

             ↓

Consistent Detection
             ↓
ALERT
```

Compare with:

```text
Frame 100 → nothing
Frame 101 → large_gun = 0.54
Frame 102 → nothing
Frame 103 → nothing
Frame 104 → nothing

             ↓

Likely False Positive
             ↓
NO ALERT
```

---

# 11. Dataset Is Critical

The dataset will probably matter more than the difference between two similar YOLO architectures.

You should collect images using the **actual thermal camera** that will be used in the project.

Example:

```text
Person 1
 ├── No weapon
 ├── Large knife concealed
 ├── Large gun concealed
 └── Explosive-device-like object concealed

Person 2
 ├── No weapon
 ├── Large knife concealed
 ├── Large gun concealed
 └── Explosive-device-like object concealed
```

---

# 12. Clothing Variations

Collect examples with many types of clothing:

```text
T-shirt
Sweatshirt
Hoodie
Thin jacket
Thick jacket
Loose clothing
Tight clothing
```

Different materials insulate heat differently.

That means the same concealed object may look different depending on the clothing.

---

# 13. Pose, Angle, and Motion

Collect:

```text
Front view
Side view
Back view
45-degree view
```

And actions such as:

```text
Standing
Walking
Turning
Sitting
Bending
```

---

# 14. Distance Variations

For example:

```text
0.5 m
1 m
2 m
3 m
4 m
```

The farther the person is from the camera:

```text
Distance ↑
      ↓
Person pixels ↓
      ↓
Weapon thermal pattern pixels ↓
      ↓
Detection becomes harder
```

This is where P2 may eventually help.

---

# 15. Thermal Timing

Also vary how long the object has been concealed.

Example:

```text
Object inserted
      ↓
Immediately captured

Object worn for 5 minutes

Object worn for 15 minutes

Object worn for 30 minutes
```

A concealed object may gradually approach the body's temperature.

That can reduce the thermal contrast.

---

# 16. Hard Negatives

This is extremely important.

Your dataset should contain many non-weapons that could look suspicious.

Examples:

```text
Phone
Wallet
Keys
Belt buckle
Power bank
Water bottle
Pen
Tools
Bag strap
Zipper
Folded clothing
Sweat
Cold clothing region
Warm clothing region
Nothing
```

Otherwise the model may learn:

```text
Cold rectangular object
      ↓
Gun
```

which would create many false positives.

---

# 17. Recommended Model Stack

| Task | Recommended Model |
|---|---|
| Person Detection | **YOLOv26s** |
| Tracking | **ByteTrack** |
| Body / Pose Estimation | **YOLOv26 Pose** |
| Concealed Object Detection | **YOLOv26m** |
| Small Concealed Patterns | **YOLOv26m + P2 head later if needed** |
| Precise Shape / Mask | **YOLOv26 Segmentation** |
| Alternative Accuracy Benchmark | **RT-DETR** |

---

# 18. Recommended MVP

Start with:

```text
Thermal Camera
      ↓
YOLOv26s
Person Detection
      ↓
ByteTrack
      ↓
Person Track
      ↓
Crop Person
      ↓
YOLOv26m
Concealed Object Detector
      ↓
Multiple-Frame Verification
      ↓
Threat / No Threat
```

Do not initially add every possible component.

First prove that your thermal data actually contains enough information to distinguish:

```text
No Threat
vs
Concealed Threat
```

Then add:

```text
Pose
↓
Body ROIs
↓
More Classes
↓
P2
↓
Segmentation
```

---

# 19. Recommended Development Order

## Stage 1 — Basic Detection

```text
Thermal Camera
      ↓
YOLOv26s
      ↓
Person
```

## Stage 2 — Concealed Object Experiment

```text
Person Crop
      ↓
YOLOv26m
      ↓
Threat / No Threat
```

The goal is to answer:

> Does the thermal sensor actually provide enough information for this problem?

## Stage 3 — Tracking

```text
YOLOv26
      ↓
ByteTrack
      ↓
Track ID
```

## Stage 4 — Temporal Verification

```text
Track ID
      ↓
Collect Predictions
      ↓
Stable Prediction?
      ↓
Alert
```

## Stage 5 — Body ROI

```text
YOLOv26 Pose
      ↓
Chest / Waist / Back / Legs
      ↓
Search Relevant Regions
```

## Stage 6 — Small Object Improvements

If the object signatures are too small:

```text
Higher-resolution person crop
      ↓
YOLOv26m
      ↓
Add P2 if necessary
```

---

# 20. Important Limitation of Thermal Imaging

Thermal imaging should be thought of as:

```text
Hidden Object
      ↓
Changes Heat Transfer
      ↓
Changes Clothing Surface Temperature
      ↓
Thermal Camera Detects Pattern
```

Not:

```text
Thermal Camera
      ↓
Directly Sees Through Clothes
```

Factors that can make detection difficult include:

- Thick clothing
- Loose clothing
- Highly insulating fabrics
- Environmental temperature
- Object temperature
- Time the object has been worn
- Distance
- Motion blur
- Camera angle

---

# 21. Better Production Sensor — Millimeter Wave

If the real requirement is:

> Detect a large knife, large gun, or explosive device beneath clothing reliably.

Then the recommended production direction is **millimeter-wave sensing**.

A stronger architecture is:

```text
                    PERSON
                      ↓
             Person Detection
                YOLOv26
                      ↓
                  ByteTrack
                      ↓
                  Track ID
                      ↓

        ┌─────────────┴─────────────┐
        ↓                           ↓
 RGB / Thermal Camera          MMWave Sensor
        ↓                           ↓
Visual / Thermal Info         Concealed Object
        ↓                           ↓
        └─────────────┬─────────────┘
                      ↓
                 Sensor Fusion
                      ↓
                Person-Level ROI
                      ↓
                   YOLOv26
                      ↓
              Concealed Threat?
                /           \
              NO            YES
                              ↓
                     Temporal Verification
                              ↓
                            ALERT
```

---

# 22. Thermal vs MMWave

## Thermal

Think:

> Does the surface temperature of the clothing look abnormal?

```text
Person
      ↓
Object affects heat flow
      ↓
Clothing temperature pattern
      ↓
Thermal detector
```

## Millimeter Wave

Think:

> Is there actually an object beneath the clothing?

```text
Person
      ↓
MMWave Sensor
      ↓
Concealed Object Signature
      ↓
YOLOv26
```

For true concealed-object detection, MMWave is the more appropriate sensor technology.

---

# 23. Recommended Overall Architecture

```text
PHASE 1
Thermal MVP
      ↓
YOLOv26s Person Detection
      ↓
ByteTrack
      ↓
YOLOv26m Concealed Object Detection
      ↓
Temporal Verification
      ↓
Evaluate if thermal information is sufficient


PHASE 2
Improve Thermal Pipeline
      ↓
YOLOv26 Pose
      ↓
Body ROIs
      ↓
Higher Resolution
      ↓
P2 if required
      ↓
Hard-negative training


PHASE 3
Production Sensor
      ↓
MMWave
      +
RGB / Thermal
      ↓
YOLOv26
      ↓
Sensor Fusion
      ↓
Temporal Verification
      ↓
Security Alert
```

---

# 24. Easy Mental Model

```text
YOLOv26s:
"I see a person."

ByteTrack:
"I will call this person Track 5."

YOLOv26 Pose:
"These are Track 5's chest, waist, and legs."

YOLOv26m:
"There is a suspicious large-gun-like pattern
around Track 5's waist."

Temporal Logic:
"I have seen the same pattern for several frames."

Application:
"Track 5 may contain a concealed threat."

→ ALERT
```

---

# Final Recommendation

For the first MVP:

```text
Thermal Camera
      ↓
YOLOv26s
      ↓
ByteTrack
      ↓
Person Crop
      ↓
YOLOv26m
      ↓
Threat / No Threat
      ↓
Multi-Frame Verification
```

Then improve with:

```text
YOLOv26 Pose
      ↓
Body ROIs
      ↓
More Classes
      ↓
P2 if needed
```

For a real production system where reliable through-clothing detection is required:

```text
MMWave
+
RGB / Thermal
+
YOLOv26
+
ByteTrack
+
Temporal Verification
```

The most important lesson is:

> **The sensor determines what information exists. YOLOv26 can only learn patterns that the sensor actually captures.**
