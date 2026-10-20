# Face Recognition Pipeline Summary

This document summarizes two pipelines:

1. **Face recognition without tracking**
2. **Face recognition with live tracking**

The example assumes:
- 1 camera
- 2 known people: `Person A` and `Person B`
- Any other face should be labeled `Unknown`

---

# 1. Pipeline Without Tracking

## High-Level Flow

```text
Camera / Image
    ↓
RetinaFace
    ↓
Face Detection + Facial Landmarks
    ↓
Face Alignment
    ↓
ArcFace
    ↓
Face Embedding
    ↓
Compare Against Known Embeddings
    ↓
Person A / Person B / Unknown
```

## Step-by-Step

### 1. Camera

The camera provides an image or video frame.

```text
Camera
   ↓
Image
```

---

### 2. RetinaFace — Face Detection

RetinaFace answers:

> **Where is the face?**

It detects:
- Face bounding box
- Eyes
- Nose
- Mouth corners

Example:

```text
┌───────────────────────────────┐
│                               │
│      ┌──────────────┐         │
│      │      🙂      │         │
│      └──────────────┘         │
│                               │
└───────────────────────────────┘
```

RetinaFace can also provide facial landmarks that are useful for alignment.

---

### 3. Face Alignment

Alignment rotates, scales, and crops the face so that facial features appear in approximately standard positions.

For example:

```text
Before:

    🙂
   /

After:

    🙂
```

This makes the input more consistent for ArcFace.

---

### 4. ArcFace — Face Recognition / Verification

ArcFace answers:

> **Whose face is this?**

More precisely, ArcFace converts the face into a numerical representation called an **embedding**.

Example:

```text
Face
 ↓
ArcFace
 ↓
[0.12, -0.45, 0.81, ..., 0.22]
```

A typical ArcFace embedding contains hundreds of numbers.

---

### 5. Compare Embeddings

Before running the live system, create reference embeddings for the known people.

```text
Person A reference photo
    ↓
RetinaFace
    ↓
Alignment
    ↓
ArcFace
    ↓
Embedding A
```

```text
Person B reference photo
    ↓
RetinaFace
    ↓
Alignment
    ↓
ArcFace
    ↓
Embedding B
```

Then compare the detected face embedding against both.

```text
Detected Face Embedding
        ↓
   Compare with
   ↙          ↘
Person A    Person B
Embedding   Embedding
```

A common comparison method is **cosine similarity**.

Example:

```text
Detected face vs Person A = 0.84
Detected face vs Person B = 0.32

→ Person A
```

If neither similarity score is good enough:

```text
→ Unknown
```

---

## Basic Pipeline Summary

```text
CAMERA
  ↓
RetinaFace
  ↓
Face Detection
  ↓
Face Alignment
  ↓
ArcFace
  ↓
Embedding
  ↓
Cosine Similarity
  ↓
Person A / Person B / Unknown
```

---

# 2. Pipeline With Live Tracking

When working with live video, recognizing the same person independently on every frame is inefficient.

Instead, add a tracker such as **ByteTrack**.

## High-Level Flow

```text
Camera
   ↓
RetinaFace
   ↓
Face Detection
   ↓
ByteTrack
   ↓
Track ID
   ↓
Collect Several Good Face Frames
   ↓
ArcFace
   ↓
Person A / Person B / Unknown
   ↓
Store Track ID → Identity
   ↓
Continue Tracking Identity Live
```

---

## Step-by-Step

### 1. RetinaFace Detects Faces

Suppose the camera sees two people.

```text
🙂             🙂
```

RetinaFace detects both faces.

---

### 2. ByteTrack Assigns Track IDs

ByteTrack answers:

> **Is this detection the same object/person that I saw in the previous frame?**

It gives each tracked object a temporary ID.

Example:

```text
Left face  → Track ID 5
Right face → Track ID 8
```

On the next frame, the people move:

```text
   🙂               🙂
```

ByteTrack tries to keep:

```text
Person on left  → Track ID 5
Person on right → Track ID 8
```

The track ID is **not the person's real identity**.

It is only a temporary key used for tracking.

---

### 3. Collect Several Good Frames

Instead of recognizing someone from one potentially bad frame, collect several good face crops.

For example:

```text
Track ID 5
   ↓
Frame 1
Frame 2
Frame 3
Frame 4
Frame 5
```

Prefer frames where the face is:
- Large enough
- Sharp
- Mostly frontal
- Not strongly occluded
- Well lit

---

### 4. ArcFace Recognizes the Track

Run the selected face crops through ArcFace.

Example:

```text
Track ID 5
    ↓
ArcFace
    ↓
Compare with Person A / Person B
    ↓
Person B
```

You now know:

```text
Track ID 5 → Person B
```

---

### 5. Store Track ID → Identity Mapping

In code, this could look like:

```python
track_to_identity = {
    5: "Person B",
    8: "Person A"
}
```

Here:

```text
5 = key
"Person B" = value
```

So when ByteTrack returns:

```python
track_id = 5
```

the application can retrieve:

```python
track_to_identity[5]
```

and get:

```text
Person B
```

---

### 6. Keep Showing the Identity Live

After recognition, you do not need to run ArcFace on every frame.

Example:

```text
Frame 10 → Track 5 → Person B
Frame 11 → Track 5 → Person B
Frame 12 → Track 5 → Person B
Frame 13 → Track 5 → Person B
```

The live display can therefore show:

```text
┌───────────────────────┐
│      Person B         │
│         🙂            │
└───────────────────────┘
```

ByteTrack maintains the track, while your application remembers which identity belongs to that track.

---

# Important: Track ID Is Temporary

Suppose Person B leaves the camera.

Earlier:

```text
Track 5 → Person B
```

If Person B comes back later, ByteTrack may assign:

```text
Track 12
```

ArcFace can recognize them again:

```text
Track 12 → Person B
```

So:

```text
Track ID ≠ Real Identity
```

Instead:

```text
ByteTrack ID = temporary video identity
ArcFace identity = actual recognized person
```

---

# Occasional Re-Verification

Tracking can make mistakes, especially when:
- People cross each other
- One person blocks another
- Someone leaves and re-enters
- Detection temporarily disappears

For example:

```text
Person A → Track 5
Person B → Track 8
```

After crossing, a tracker could accidentally switch IDs.

Therefore, a stronger system occasionally runs ArcFace again.

```text
Track 5 = Person A
      ↓
Keep tracking
      ↓
Occasionally run ArcFace
      ↓
Still Person A?
```

This helps detect and correct tracking errors.

---

# Recommended MVP for 2 People + 1 Camera

## Without Tracking

Use this first if you only want to prove face recognition works:

```text
Camera
  ↓
RetinaFace
  ↓
Alignment
  ↓
ArcFace
  ↓
Compare Embeddings
  ↓
Person A / Person B / Unknown
```

---

## With Tracking

For the live system:

```text
Camera
  ↓
RetinaFace
  ↓
ByteTrack
  ↓
Track ID
  ↓
Collect Several Good Frames
  ↓
ArcFace
  ↓
Person A / Person B / Unknown
  ↓
Store:
Track ID → Identity
  ↓
Continue Showing Identity Live
  ↓
Occasional ArcFace Re-Verification
```

Example:

```text
Camera
   ↓
RetinaFace
   ↓
Two Faces Detected
   ↓
ByteTrack
   ↓
Track 5          Track 8
   ↓                ↓
ArcFace          ArcFace
   ↓                ↓
Person B         Person A

Store:

Track 5 → Person B
Track 8 → Person A
```

Then:

```text
Frame after frame:

Track 5 → Person B
Track 8 → Person A
```

---

# Where DeepFace Fits

**DeepFace is a framework, not the recognition model itself.**

It can help connect different components.

For example:

```text
DeepFace
   ├── RetinaFace → Face Detection
   ├── Face Alignment
   ├── ArcFace → Face Embedding
   └── Similarity / Verification
```

For an initial prototype:

```text
DeepFace + RetinaFace + ArcFace
```

is a reasonable choice.

For a live tracking application, the video loop and ByteTrack logic would normally be handled separately:

```text
OpenCV Camera
    ↓
RetinaFace
    ↓
ByteTrack
    ↓
DeepFace / ArcFace
    ↓
Identity Mapping
```

---

# What Each Component Does

| Component | Purpose | Simple Question |
|---|---|---|
| **RetinaFace** | Face detection + landmarks | Where is the face? |
| **Face Alignment** | Normalize face orientation | Can I make the face straight and consistent? |
| **ArcFace** | Face embedding / recognition | Whose face is this? |
| **ByteTrack** | Track detections across frames | Is this the same face/object as before? |
| **DeepFace** | Framework that connects face models | Can I easily run the pipeline? |
| **Track ID** | Temporary tracking key | Which tracked object is this? |

---

# Easy Mental Model

Think of a security guard:

```text
RetinaFace:
"I see a face over there."

ByteTrack:
"That is the same person I was following a second ago.
I'll call them Track 5."

ArcFace:
"Track 5's face matches Person B."

Application:
"Okay, from now on:

Track 5 → Person B"
```

Final live result:

```text
┌─────────────────────────────┐
│                             │
│  Person A       Person B    │
│     🙂              🙂      │
│                             │
└─────────────────────────────┘
```


---

# Recommended Camera for the MVP

For the first prototype with:

- 1 camera
- 2 people
- RetinaFace
- ByteTrack
- ArcFace

you do **not** need an expensive camera.

## Cheap Starting Option

### Logitech C270

A simple **Logitech C270 USB webcam** is a good low-cost starting point.

Typical capabilities:

- 720p resolution
- 30 FPS
- USB connection
- Works easily with OpenCV
- Common and relatively easy to find in Israel

For an MVP where people are roughly **1–2 meters from the camera**, it should be enough to test the complete pipeline:

```text
Logitech C270
      ↓
RetinaFace
      ↓
ByteTrack
      ↓
ArcFace
      ↓
Person A / Person B / Unknown
```

## When to Upgrade

The main limitation of a 720p camera is that faces contain fewer pixels when people are farther away.

If the initial prototype works and you need better recognition quality, move to a basic **1080p USB webcam**, such as a Logitech C920/C922-class camera or similar.

```text
MVP / Proof of Concept
        ↓
Logitech C270
720p / 30 FPS
        ↓
Test real distance + lighting
        ↓
Need more facial detail?
       / \
     No   Yes
     ↓      ↓
 Keep     Upgrade to
 C270     1080p webcam
```

## OpenCV Example

A normal USB webcam is easy to access using OpenCV:

```python
import cv2

cap = cv2.VideoCapture(0)

while True:
    ret, frame = cap.read()

    if not ret:
        break

    cv2.imshow("Camera", frame)

    if cv2.waitKey(1) == ord("q"):
        break

cap.release()
cv2.destroyAllWindows()
```

## Camera Recommendation Summary

| Stage | Recommendation |
|---|---|
| First MVP | Logitech C270 |
| Resolution | 720p |
| FPS | 30 FPS |
| Connection | USB |
| Expected use | 2 people, one fixed camera |
| Upgrade path | Basic 1080p webcam |
| Important factors | Camera distance, lighting, face size in the frame |

For this project, **camera placement and lighting can matter more than buying a very expensive camera**. Start cheap, test the actual environment, and upgrade only if the face crops are too small or unclear for reliable recognition.


---

# Future Camera Upgrade

Once the MVP works reliably with the Logitech C270, a good next step is to upgrade to a **1080p webcam**.

## Recommended Future Option

### Logitech C920 / C922 Class

A Logitech C920/C922-class webcam is a strong practical upgrade because it gives you:

- 1080p resolution
- Better facial detail
- Better image quality than basic 720p webcams
- USB connection
- Easy OpenCV integration
- Better performance when people are slightly farther from the camera

This can improve the quality of the face crop that RetinaFace sends into ArcFace.

```text
1080p Camera
    ↓
RetinaFace
    ↓
Larger / clearer face crop
    ↓
Face Alignment
    ↓
ArcFace
    ↓
More reliable embedding
    ↓
Person A / Person B / Unknown
```

## Why 1080p Helps

For face recognition, the important part is not only the full camera resolution.

What really matters is:

> **How many pixels does the person's face occupy in the image?**

For example:

```text
720p frame

Person far away
     ↓
small face crop
     ↓
less facial detail
```

Compared with:

```text
1080p frame

Same person / same distance
     ↓
larger face crop
     ↓
more facial detail
     ↓
better input for ArcFace
```

## Suggested Upgrade Path

```text
Stage 1
Logitech C270
720p / 30 FPS
    ↓
Build MVP
    ↓
Test RetinaFace + ByteTrack + ArcFace
    ↓
Measure recognition quality
    ↓
Need more detail?
   /       \
 No         Yes
 ↓           ↓
Keep C270   Upgrade to 1080p
             ↓
       Logitech C920/C922 class
```

## Camera Roadmap

| Project Stage | Camera | Why |
|---|---|---|
| Initial MVP | Logitech C270 | Cheap and sufficient for proving the pipeline |
| Improved prototype | Logitech C920/C922 class | 1080p and better facial detail |
| Future production system | Higher-quality 1080p/4K camera chosen for the real environment | Better optics, lighting performance, distance, and reliability |

## Important for Future Production

Before buying an expensive production camera, test the real installation conditions:

- Distance between people and camera
- Indoor/outdoor lighting
- Camera angle
- Number of people visible at once
- Face size in pixels
- Motion blur
- Low-light conditions
- Whether people are looking toward the camera
- Required recognition accuracy

A more expensive camera does not automatically solve recognition problems.

For this pipeline, a **clear, sufficiently large face crop** is usually more important than simply buying the highest-resolution camera available.
