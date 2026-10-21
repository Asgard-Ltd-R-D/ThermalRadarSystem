# A 24° thermal lens for the PoC; Thermal Verification covers about 1.5–3 m

The projection research recommended a lens of about 50° (`docs/research/thermal-roi-projection.md`). For the PoC we chose the Boson+ 640 with the 24° AS24 lens instead: part 22640AS24-6IARX (`docs/research/boson-plus-640.md`). Two reasons:

- The PoC room is 5×7 m, larger than the customer's room of about 2×5 m.
- The choice depends on what FLIR has in stock.

The 32° AS32 (22640AS32-6IARX) may replace it, depending on stock.

## The range limit

The Radar System builds Suspect Regions only for Persons 0.5–3 m away (`docs/research/pulse72-radar-system.md`). So the extra reach a narrow lens offers in a bigger room goes unused unless that limit is raised.

## What each lens gives within 0.5–3 m

| Lens | Torso fits from | Width seen at 3 m (room is 5 m) | Region at 3 m | Blur at 1.5 m and 3 m, focused at 2 m |
|---|---|---|---|---|
| 24° AS24 (chosen) | 1.5 m | 1.29 m | 82 px | ≈ 4 px |
| 32° AS32 (possible swap) | 1.1 m | 1.72 m | 62 px | ≈ 2.5 px |
| 50° AS50 (research pick) | 0.7 m | 2.79 m | 42 px | ≈ 1 px |

All three come as catalogued radiometric parts, so radiometry doesn't decide between them.

## Consequences

- **The verification zone is about 1.5–3 m** along the path. A straight walk gives about 1 s, or 32 frames, in it. Outside the zone, "not visible" is a normal outcome, not a fault.
- **The lens must be refocused to about 2 m before calibration or capture.** As delivered, the AS24 is sharp only beyond 33 m.
- **Side positions are covered last.** Off-centre Persons, and the second of two side-by-side Persons, lose coverage first.
- **If the Radar System's range is raised for the PoC room,** the 24° lens pays off: it gives 35 px at 7 m, against 27 px for the 32° and 18 px for the 50°.
- **The camera tilts to the torso band** rather than staying parallel to the Hanwha.
- **The lens is hard to swap once data collection starts,** because thermal training data is tied to it. Decide between 24° and 32° before then.
