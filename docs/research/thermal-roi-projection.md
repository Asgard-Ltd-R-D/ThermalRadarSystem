# Mapping a Suspect Region into thermal image coordinates (research, 2026-10-20)

Answers [#4](https://github.com/Asgard-Ltd-R-D/ThermalRadarSystem/issues/4). Code facts come from a read-only look at pulse72 (`~/Documents/DeviceDetection`, branch `pipeline` @ `9cc5288`; see the [pulse72 investigation][P72]). Numbers marked _(assumption)_ are inputs I chose, not measurements.

## Answer

**For v1, use (a): project the Suspect Region in 3D through the thermal intrinsics and a thermal-to-car extrinsic.** The 3D region should be the region's RGB window back-projected to the depth of its radar points. It should not be the raw radar points.

- Get thermal-to-car by chaining two transforms: one board calibration of thermal↔RGB, plus the Radar System's existing RGB↔car transform. The RGB↔car error then cancels out, because the corners were built through the same transform.
- The expected error is about 10–15% of the region's width at 0.5–3 m. A plain homography (b) is off by 57 px at 10 cm baseline, which is 1.5× the region's width at 3 m.
- Person-level matching (c) fails at close range: at 0.5 m a 25–50° lens sees only 0.22–0.47 m of the Person.

## What the Radar System has today

- **The region is an RGB-pixel box.** A Suspect Region is a sliding window over the Person's torso in raw distorted fisheye pixels, `cx, cy, w, h` (`weapon_detector.py:231-245`).
  - The window is 55% of the torso height tall and 60% of that wide, and needs ≥ 15 radar points (`models/config.py:22-26`).
  - Training labels are RGB boxes too: a window is positive when its IoU with the labelled box is ≥ 0.3 (`models/config.py:27`).
  - So the window defines *where* the region is, and the radar points are its features.
- **The 3D data exists but is not carried out.** Every radar point in the window has car-frame `x, y, z`: the projected DataFrame keeps them (`weapon_detector.py:752-762`). `_points_in_windows` returns only `uv_x, uv_y, power` per window (`models/realtime.py:577-598`). The Person range gate is the median car `y` in 0.5–3 m (`models/config.py:10-11`).
- **The RGB projection is the OpenCV fisheye model.** `project_points` calls `cv2.fisheye.projectPoints` with `R, t` from `build_extrinsic` (`radar_overlay.py:421-520`).
  - Intrinsics: K = [930, 930, 960, 540] and k1–k4 = −0.826, 3.134, −4.795, 2.554 (`Hanwha.cam`).
  - Camera position in the car frame: (−0.072, 0, +0.1) m. The `.cam` file stores this as [right, up, depth] (`radar_overlay.py:292-298`).
- **The radar↔RGB rotation is not calibrated.** The fine angles `cam_alpha/beta/gamma` are 0 (`launch_config_sys1.yaml:145-156`) and can only be nudged by keyboard in 1° steps (`radar_overlay.py:2541,2736-2746`). At the Hanwha's f = 930 px, 1° is about 16 px.
- **Reusable calibration code already exists.**
  - `radar_overlay.py:1141-1168` undistorts fisheye corners and then runs `cv2.solvePnP` with zero distortion.
  - `legacy/cam_calib/calibration/charuco_calibrate.py` is a ChArUco calibration script.
  - `legacy/radar/calibration_utils.py` fits radar↔camera from clicked pixel/radar-xyz pairs.
- **Points nearer than 0.5 m are dropped.** `project_points` discards anything closer than `MIN_DEPTH_M = 0.5` m of camera depth (`radar_overlay.py:207,489`), so a Person at 0.5 m is at the hard edge.

## Worked geometry

Reference thermal camera: 640×512 with 50° HFOV, which gives f = 320/tan 25° ≈ 686 px. The RGB↔thermal baseline is B = 10 cm _(assumption: "mounted near")_. The Suspect Region is 0.165 × 0.275 m _(assumption: 0.5 m shoulder-to-hip, times the window ratios above)_.

| Depth Z | 0.5 m | 1 m | 2 m | 3 m |
|---|---|---|---|---|
| Region width, 640 @ 50° (f 686) | 226 px | 113 px | 57 px | 38 px |
| Region width, 320 @ 50° (f 343) | 113 px | 57 px | 28 px | 19 px |
| Scene seen, 50° lens (W×H) | 0.47×0.37 m | 0.93×0.75 m | 1.87×1.49 m | 2.80×2.24 m |
| Scene seen, 25° lens (W×H) | 0.22×0.18 m | 0.44×0.35 m | 0.89×0.71 m | 1.33×1.06 m |

Error sensitivities at f = 686 px. These follow from pinhole two-view geometry, where disparity = f·B/Z and a plane-induced homography is exact only for points on that plane ([HZ] §13):

| Error source | Pixel error | 0.5 m | 1 m | 3 m |
|---|---|---|---|---|
| Homography fixed at plane Z0 = 0.86 m (minimax over 0.5–3 m), B = 10 cm | f·B·\|1/Z − 1/Z0\| | 57 | 11 | 57 |
| Same, B = 5 cm | | 29 | 6 | 29 |
| Depth error δZ = 10 cm when back-projecting, B = 10 cm | f·B·δZ/Z² | 27 | 6.9 | 0.8 |
| Extrinsic rotation error, per 0.1° | f·δθ | 1.2 | 1.2 | 1.2 |
| Extrinsic translation error, per 1 cm | f·δt/Z | 14 | 6.9 | 2.3 |
| Time skew, per 10 ms at 1 m/s lateral | f·v·Δt/Z | 14 | 6.9 | 2.3 |

## The three options

### (a) Project 3D through thermal intrinsics and thermal-to-car extrinsics (recommended)

- **Inputs from the Radar System:** the region in 3D in the car frame and the frame timestamp.
  - Use the window's four corners, back-projected to the median car `y` of the window's points: `cv2.fisheye.undistortPoints` gives a ray, which is scaled to that depth and moved to the car frame.
  - Do not use the raw points. Their hull is sparse (≥ 15 points), and it inherits the radar↔RGB misregistration: the label lives in the window, not in the points.
- **Projection:** call `cv2.projectPoints(corners_car, rvec_th, t_th, K_th, D_th)` and take the bounding rectangle, clipped to the image ([CV] L855).
- **Error, two ways to get thermal-to-car:**
  - **Chain through RGB (recommended):** T_th←car = T_th←RGB · T_RGB←car. The corners were built through the same T_RGB←car, so its error cancels and only these terms are left:
    - the depth term, f·B·δZ/Z²;
    - the thermal↔RGB calibration error;
    - time skew.
    - Example _(assumption: 0.2°, 5 mm, δZ 10 cm, 10 ms)_, combined as a root-sum-square: about 31 px at 0.5 m (14% of the width) and 3.6 px at 3 m (9%).
  - **Calibrate radar↔thermal directly:** Domhof et al. report a median camera-to-radar RMSE of about 2.5 cm with ≥ 10 board positions, using a trihedral reflector behind Styrofoam on a radar without elevation [DOM]. That is 34 px at 0.5 m and 5.7 px at 3 m, about 15% at both ends. It is worse at close range and takes more effort, so defer it.
- **Calibration:** thermal intrinsics, plus one thermal↔RGB board session (procedure below).

### (b) Map RGB-fisheye pixels to thermal pixels

- **A homography needs undistorted images.** `cv2.findHomography` "finds a perspective transformation between two planes" [CV L678], so the fisheye pixels must first go through `cv2.fisheye.undistortPoints`. The raw fisheye model, θd = θ(1 + k1θ² + … + k4θ⁸), bends straight lines, so no homography exists on raw pixels [CV L405-437].
- **Plain homography (needs only the window's pixel box):** the parallax error is 57 px at both 0.5 m and 3 m with B = 10 cm, and 29 px with B = 5 cm. That is 75–150% of the region's width at 3 m.
  - To stay within 10% of the width at 3 m, the baseline must be ≤ 7 mm, which effectively means a beam-splitter or co-boresighted rig.
  - **Rejected.**
- **Depth-dependent mapping (needs the box plus the region's depth):** back-project the RGB pixel at the radar depth, transform it to the thermal frame, and project it. PST900 does exactly this with stereo depth ([PST] eq. 1–2). The arithmetic and calibration are identical to (a) chained through RGB, so (a) and (b) converge. Carrying the geometry in the car frame keeps the fisheye model out of the Thermal Verifier.

### (c) Match at Person level only

- **Inputs:** the Person's track ID and box, which the Radar System computes but does not carry (`pose_people`; [P72]), plus a thermal Person detector to match against.
- **Error:** none at region level, because the region is never located. The Thermal Verifier would have to judge the whole Person. The region is about 5% of a 0.5 × 1.7 m Person _(assumption)_.
- **Fails up close.** At 0.5 m the lens sees 0.22–0.47 m, so the Person does not fit in the frame.
- **Calibration:** none, or a rough bearing alignment.
- **Verdict:** the cheapest calibration, but it discards the Radar System's localisation. Use it only as a fallback when no extrinsic is available.

## v1 calibration procedure and effort

1. **Board.** Use one board visible in both RGB and LWIR without heating.
   - PST900 uses sand-blasted aluminium squares on black acrylic. The aluminium reflects the operator's heat, and the same corners are detected in both modalities [PST §III-A].
   - A high-emissivity UV print on polished stainless steel (ε 0.24 vs 0.93) reached a mean reprojection error of 0.057 px on a 640×480 LWIR camera [ELS].
   - Heated paper is the fallback: 0.41–1.27 px in the literature, and it cools "in a few minutes" ([ELS] Table 1). A hole mask over a heat source cut reprojection error by up to 78% compared with a heated chessboard [VID].
   - I found no first-party vendor application note on thermal geometric calibration.
2. **Thermal intrinsics.** Take 20–40 views spread across the frame. Detect corners with `cv2.findChessboardCornersSB` [CV L1336]. PST900 found OpenCV's detector poor on thermal images and used libcbdetect.
   - Then run `cv2.calibrateCamera` with the pinhole and radial-tangential model for a 25–50° lens. PST900 used the fisheye model for its wide 2.3 mm Boson lens.
   - _Effort: about half a day once the board exists (estimate)._
3. **Thermal↔RGB extrinsics.** Take about 15 simultaneous captures with the board at 0.5–3 m. OpenCV cannot calibrate a fisheye+pinhole pair directly:
   - `fisheye::stereoCalibrate` takes 4-term fisheye distortion for both cameras [CV L4010-4060].
   - 5.x `calibrateMultiview` "does not support mix of different camera models" [CV5 L1054].
   - So undistort the RGB corners first, as `radar_overlay.py:1141-1168` already does. Then run `cv2.stereoCalibrate` with zero RGB distortion and `CALIB_FIX_INTRINSIC`, which OpenCV recommends when the intrinsics are known [CV L1850-1856]. It returns R, T from RGB to thermal [CV L1753-1759].
   - _Effort: about half a day (estimate)._
4. **Compose.** Compute T_th←car = [R|T] · `build_extrinsic(t_in_car, euler)`, and store it with the RGB↔car calibration it was built from.
5. **Check.** Place a warm object at 0.5, 1.5 and 3 m and compare its projected box with its thermal blob. The click-pair format in `legacy/radar/calibration_utils.py:927` can log these pairs.

**Libraries:** OpenCV 4.11 and numpy, both already in the container [P72]. No new dependency is needed.

## Fields the Radar System's output must carry (feeds #7, #9)

These fields are per Suspect Region:

| Field | Status | Notes |
|---|---|---|
| `stamp` | exists per frame | Camera frame time on the host clock (`radar_overlay.py:1582`). The Thermal Verifier picks the nearest thermal frame; each 10 ms of skew costs 14 px at 0.5 m. |
| `person_track_id`, `region_track_id` | exist (`track_id`, `weapon_track_id`) | |
| `corners_car_m` | **new** | 4 × (x, y, z) in metres in the car frame: the window corners back-projected to `depth_m`. |
| `depth_m` | **new** | Median car `y` of the window's radar points. The value is already computable from the projected DataFrame. |
| `calib_id` | **new** | Identifies the RGB↔car calibration (`.cam` file plus Euler angles) the corners were built with. The thermal extrinsic is composed from it and is valid only with it. |
| `cx, cy, w, h` | exists | Fisheye pixels, for display only. |

The Thermal Verifier also needs a "not visible" outcome. At 0.5 m a 25° lens sees 0.22×0.18 m, which is smaller than the 0.165×0.275 m region, and at 3 m a 320-wide 50° camera puts the region in only 19×31 px.

## Sources

- [P72]: [pulse72 investigation](pulse72-radar-system.md), and pulse72 `~/Documents/DeviceDetection` @ `9cc5288`, read-only; file:line references are cited inline.
- [CV]: OpenCV 4.11.0, [`modules/calib3d/include/opencv2/calib3d.hpp`](https://github.com/opencv/opencv/blob/4.11.0/modules/calib3d/include/opencv2/calib3d.hpp). This is the source of the [fisheye](https://docs.opencv.org/4.11.0/db/d58/group__calib3d__fisheye.html) and [calib3d](https://docs.opencv.org/4.11.0/d9/d0c/group__calib3d.html) docs. Line numbers are cited inline.
- [CV5]: OpenCV 5.x, [`modules/calib/include/opencv2/calib.hpp`](https://github.com/opencv/opencv/blob/5.x/modules/calib/include/opencv2/calib.hpp), `calibrateMultiview`.
- [ELS]: ElSheikh et al., "Infrared Camera Geometric Calibration: A Review and a Precise Thermal Radiation Checkerboard Target", *Sensors* 23(7):3479, 2023. <https://pmc.ncbi.nlm.nih.gov/articles/PMC10099184/>
- [VID]: Vidas et al., "A Mask-Based Approach for the Geometric Calibration of Thermal-Infrared Cameras", *IEEE TIM* 61(6):1625–1635, 2012. <https://openresearch-repository.anu.edu.au/handle/1885/24551>
- [PST]: Shivakumar et al., "PST900: RGB-Thermal Calibration, Dataset and Segmentation Network", ICRA 2020. <https://arxiv.org/abs/1909.10980>
- [DOM]: Domhof, Kooij and Gavrila, "A Joint Extrinsic Calibration Tool for Radar, Camera and Lidar", *IEEE T-IV* 6(3):571–582, 2021, doi:10.1109/TIV.2021.3065208.
- [HZ]: Hartley and Zisserman, *Multiple View Geometry in Computer Vision*, 2nd ed., ch. 13 (homography induced by a plane).
- Radar accuracy for context: the Arbe Phoenix spec gives 0.7° azimuth / 0.9° elevation resolution and 4–20 cm range accuracy (<https://arberobotics.com/product/>). The δZ = 10 cm assumption sits inside that range.
