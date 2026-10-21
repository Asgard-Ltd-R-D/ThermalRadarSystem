# FLIR Boson+ 640: facts for v1 (research, 2026-10-21)

Supports [Which thermal camera does v1 target?](https://github.com/Asgard-Ltd-R-D/ThermalRadarSystem/issues/5). Facts come from FLIR's own documents. **[mirror]** marks a FLIR-written document fetched from a reseller's copy because FLIR gates it behind a login. Values marked _(derived)_ are my arithmetic, not FLIR's.

## What v1 orders

- **Camera:** Boson+ 640, shuttered (part numbers starting 22640), Industrial grade, radiometric, with the **A024** lens.
  - The **A032** lens may replace it, depending on stock: 14 mm, 32.0° × 25.6°, f/1.00, f ≈ 1167 px _(derived)_, radiometric-capable [DS117 Table 11; KB-OPT].
- **Interface:** the FLIR **USB VPC kit 421-0061-00**.
- **Confirm in writing on the quote:**
  - the exact radiometric A024 part number;
  - the ECCN and any export licence (see [Frame rate](#frame-rate-and-export-class)).

## Sensor and lens

- 640×512, 12 µm pitch. Industrial NETD is ≤ 20 mK, Professional ≤ 30 mK (lensless, f/1.0) [DS117 Table 1, 13].
- **A024:** 18 mm focal length, 24.3° × 19.5° FOV, f/1.04 [DS117 Table 11].
  - f ≈ 1500 px _(derived: 18 mm / 12 µm)_. Then 2·atan(320/1500) = 24.1°, which matches the stated FOV, so distortion is low.
  - FLIR's optics KB lists A024 as radiometric-capable for 640 [KB-OPT].
- **The 50° AS50 lens, considered and not chosen:** 9.2 mm, 49.9° × 39.3°, distortion < 18%.
  - Its radiometric support is contradicted: the product page lists an AS50 radiometric model, but the KB doesn't [KB-OPT].
- Radiometry is a separate factory SKU ("R" in the part number). A fielded camera can't be upgraded [DS117 §6.2, §6.15].
  - T-Linear output is Kelvin × 100 in high gain, so 20 °C reads 29315.
  - Accuracy is ±5 °C [DS117 §11.5].

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

- whether 60 Hz Y16 works over USB2;
- the exact Boson+ FFC freeze time;
- the telemetry timestamp's epoch;
- the ECCN;
- the radiometric A024 part number;
- the Rev410 licence text.

## Sources

- [DS117]: Boson+ Product Datasheet 102-2013-45 Rel. 117, 16 Sep 2026. <https://flir.netx.net/file/asset/55485/original/attachment/>, linked from <https://oem.flir.com/products/boson-plus?vertical=lwir&segment=oem>
- [KB-OPT]: Boson+ optical specs. <https://oem.flir.com/support/support-center/knowledge-base/boson-plus-optical-specs/>
- [KB-UVC]: <https://oem.flir.com/support/support-center/knowledge-base/what-is-the-serial-command-to-select-a-uvc-video-stream-format-ir16-or-colorized/>
- [BosonUSB]: <https://github.com/FLIR/BosonUSB>
- [rawBoson]: <https://github.com/FLIR/rawBoson>
- [FFC-AN]: Boson FFC/NUC app note (2018). <https://flir.netx.net/file/asset/16444/original/attachment/>
- [IDD] **[mirror]**: Boson+ User SDK Documentation rev410. <https://groupgets-files.s3.amazonaws.com/Boson_Plus_CZ/Boson%20Plus_User_SDK_Documentation_rev410.pdf>
