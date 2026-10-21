# FLIR Boson+ 640: facts for v1 (research, 2026-10-21)

Supports [Which thermal camera does v1 target?](https://github.com/Asgard-Ltd-R-D/ThermalRadarSystem/issues/5). Facts come from FLIR's own documents. **[mirror]** marks a FLIR-written document fetched from a reseller's copy because FLIR gates it behind a login. Values marked _(derived)_ are my arithmetic, not FLIR's.

## What v1 orders

- **Camera:** **22640AS24-6IARX**: Boson+ 640, 24° compact lens, shuttered, Industrial grade, radiometric [MF].
  - The **22640AS32-6IARX** (32° compact lens, radiometric) may replace it, depending on stock [MF].
- **Interface:** the FLIR **USB VPC kit 421-0061-00**.
- **Confirm in writing on the quote:**
  - that the part number delivered is radiometric (see [Radiometry](#radiometry));
  - the ECCN and any export licence (see [Frame rate](#frame-rate-and-export-class)).

## Sensor and lens

- 640×512, 12 µm pitch. Industrial NETD is ≤ 20 mK, Professional ≤ 30 mK (lensless, f/1.0) [DS117 Table 1, 13].
- **Lens options:**

| Lens | Focal length | FOV (H × V) | f/# | Distortion | f in px _(derived)_ | Radiometric SKU [MF] |
|---|---|---|---|---|---|---|
| **AS24 (chosen)** | 18 mm | 24.3° × 19.2° | 1.09 | < 2% | 1500 | 22640AS24-6IARX |
| AS32 (possible swap) | 13.6 mm | 32.0° × 25.6° | 1.01 | < 3% | 1133 | 22640AS32-6IARX |
| AS50 (research pick) | 9.2 mm | 49.9° × 39.3° | 1.01 | < 18% | 767 | 22640AS50-6IARX |

  Sources: [KB-OPT], [DS117 Table 11].
- For the AS24, 2·atan(320/1500) = 24.1°, which matches the stated 24.3°. A pinhole model is a fair nominal.
- The full-size A024 (18 mm) and A032 (14 mm) share these fields of view, but the model finder lists them as **non-radiometric**.

## Radiometry

- **Radiometry is a separate factory SKU,** marked by "R" in the part number. A fielded camera can't be upgraded [DS117 §6.2, §6.15].
- **FLIR's two lists disagree on which lenses come radiometric:**
  - The **model finder** sells radiometric 640s only as `-6IARX` parts with the AS24, AS32, AS50 and AS95 compact lenses, plus the A018. It lists the A024 and A032 as non-radiometric [MF].
  - The **optics KB** asterisks only A024 and A032 as "available radiometry capable", and not the AS lenses [KB-OPT].
- **Order the catalogued `-6IARX` part, and confirm on the quote that it is radiometric.**
- **Output:** T-Linear values are Kelvin × 100 in high gain, so 20 °C reads 29315.
- **Accuracy:** ±5 °C [DS117 §11.5].

## Focus

- **As delivered, the lens is focused far away, and the verification zone is out of focus.** The good-focus limits from FLIR's Note 3 [KB-OPT]:

| Lens | Closest sharp distance, new camera | Closest sharp distance, after refocusing |
|---|---|---|
| AS24 | 33 m | 1.0 m |
| AS32 | 20 m | 0.5 m |
| AS50 | 7.5 m | 0.3 m |

- **Refocus to about 2 m before calibration or any capture,** then lock the focus.
  - Refocusing changes the intrinsics, so the camera must be recalibrated after any focus change.
  - FLIR requires a focus tool for focal lengths under 18 mm, which includes the AS32 and AS50. The AS24 is at exactly 18.0 mm.
- **Depth of field across 1.5–3 m, with focus at 2 m** _(derived; thin-lens model)_:
  - The AS24 blurs by about 50 µm (≈ 4 px) at 1.5 m and at 3 m.
  - The AS32 blurs by about 31 µm (≈ 2.5 px).
  - The AS50 blurs by about 14 µm (≈ 1 px).
  - For comparison, LWIR diffraction alone is about 2 px at f/1.

## Frame rate and export class

- Every Boson+ is a **60 Hz** part (frame-rate code `6`). There is no 9 Hz Boson+.
- Output rate is user-selectable: 60, 30, 20, 15, 12, 10, 8.6 or 4.3 Hz [DS117 Table 1]. 30 Hz comes in one of two ways:
  - **Averager:** two sensor frames averaged. NETD is about 20% lower. A change needs a save plus a power cycle [DS117 §5.1, §7.3].
  - **Frame skip `fs=1`:** output rate is 60/(fs+1), with no averaging [DS117 §5.2, Table 7]. **v1 uses this.**
- **ECCN: FLIR contradicts itself.**
  - The family page says 6A003.b.4.a.
  - A model page says 6A003.b.4.b (fast) / 6A993.a (slow).
  - Microbolometer arrays map to 6A003.b.4.**b** under 15 CFR 774. Setting 30 Hz in software does not change the maximum rate _(derived reading, not FLIR's)_.

## USB output

- **UVC-compliant**, captured with plain V4L2 and no vendor video driver [DS117 §8.2.3; [BosonUSB], MIT].
  - The format is chosen by the capture software, not by a camera command [KB-UVC].
  - **`Y16`** is the 16-bit pre-AGC stream (IR16, T-Stable or T-Linear), 640×512, or 640×514 with telemetry.
  - The other formats are colorized YCbCr only. There is no 8-bit mono format.
- The control channel is a serial port, `/dev/ttyACM0` at 921600 baud [rawBoson].
- **The VPC kit is USB2 only** [DS117 §15.1]. Y16 at 30 Hz is about 19.7 MB/s, which fits. At 60 Hz it is about 39.5 MB/s, which FLIR never confirms works over USB2 _(derived)_.
- **Check on arrival:**
  - the part number is the radiometric one;
  - refocus to about 2 m (see [Focus](#focus));
  - `v4l2-ctl --list-formats-ext`;
  - `lsusb -v` for the endpoint type;
  - the telemetry frame counter, for drops.
- **Connector:** Hirose DF40C-80DP-0.4V(51), so an adapter board is needed. Mount the camera with the connector above the optical centre, or the image comes out upside down [DS117 §4].

## Telemetry and timing

- Request 640×514 to get the telemetry line, which is on by default [DS117 §6.7, Table 5]. It carries:
  - a 32-bit frame counter that does not advance on duplicated frames;
  - a millisecond timestamp with **no stated epoch**;
  - FFC "imminent" and "in progress" bits;
  - temperatures.
- External sync (`EXT_SYNC`, pin 72, 1.8 V) exists on the camera. The USB VPC kit doesn't expose it, and v1 doesn't need it.

## Flat-field correction (FFC)

- **Video freezes during FFC, repeating the last frame.** The telemetry line keeps updating [DS117 §7.9].
  - Duration: "less than a second" [DS117 §6.6]. The original Boson's app note says about 0.5 s [FFC-AN].
- **Defaults on shuttered units:**
  - automatic mode;
  - every 300 s, plus on a temperature delta (the default is stated inconsistently in DS117);
  - a 2 s "imminent" warning.
- Manual mode runs FFC only on command (`bosonRunFFC()`). There is no hardware trigger pin.

## SDK

- Rev410 provides C, C# and Python bindings, but for **camera control only**; video goes through V4L2 [IDD **[mirror**]].
- The licence is proprietary: internal use only, no redistribution, no linking with copyleft code [rawBoson].
- v1 avoids the SDK. Settings are written once with FLIR's tool and saved as power-on defaults.

## Not verified from a FLIR source

- which of FLIR's two radiometry lists is right;
- whether 60 Hz Y16 works over USB2;
- the exact Boson+ FFC freeze time;
- the telemetry timestamp's epoch;
- the ECCN;
- the Rev410 licence text.

## Sources

- [DS117]: Boson+ Product Datasheet 102-2013-45 Rel. 117, 16 Sep 2026. <https://flir.netx.net/file/asset/55485/original/attachment/>
- [MF]: Boson+ model finder (product page). <https://oem.flir.com/products/boson-plus/?vertical=lwir&segment=oem>
- [KB-OPT]: Boson+ optical specs, Notes 1 and 3. <https://oem.flir.com/support/support-center/knowledge-base/boson-plus-optical-specs/>
- [KB-UVC]: <https://oem.flir.com/support/support-center/knowledge-base/what-is-the-serial-command-to-select-a-uvc-video-stream-format-ir16-or-colorized/>
- [BosonUSB]: <https://github.com/FLIR/BosonUSB>
- [rawBoson]: <https://github.com/FLIR/rawBoson>
- [FFC-AN]: Boson FFC/NUC app note (2018). <https://flir.netx.net/file/asset/16444/original/attachment/>
- [IDD] **[mirror]**: Boson+ User SDK Documentation rev410. <https://groupgets-files.s3.amazonaws.com/Boson_Plus_CZ/Boson%20Plus_User_SDK_Documentation_rev410.pdf>
