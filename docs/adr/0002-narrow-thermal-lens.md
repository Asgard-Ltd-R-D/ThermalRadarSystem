# A 24° thermal lens for the PoC; Thermal Verification covers about 1.5–3 m

The projection research recommended a lens of about 50° (`docs/research/thermal-roi-projection.md`). For the PoC we chose the Boson+ 640's 24° A024 lens instead (`docs/research/boson-plus-640.md`). Two reasons:

- The PoC room is 5×7 m, larger than the customer's room of about 2×5 m.
- The choice depends on what FLIR has in stock.

The 32° A032 lens may replace the A024, depending on stock.

## The range limit

The Radar System builds Suspect Regions only for Persons 0.5–3 m away (`docs/research/pulse72-radar-system.md`). So the extra reach a narrow lens offers in a bigger room goes unused unless that limit is raised.

## What each lens gives within 0.5–3 m

| Lens | Torso fits from | Width seen at 3 m (room is 5 m) | Region at 3 m | Radiometric |
|---|---|---|---|---|
| 24° A024 (chosen) | 1.5 m | 1.29 m | 82 px | FLIR confirms |
| 32° A032 (possible swap) | 1.1 m | 1.72 m | 64 px | FLIR confirms |
| 50° AS50 (research pick) | 0.7 m | 2.79 m | 42 px | FLIR contradicts itself |

## Consequences

- **The verification zone is about 1.5–3 m** along the path. A straight walk gives about 1 s, or 32 frames, in it. Outside the zone, "not visible" is a normal outcome, not a fault.
- **Side positions are covered last.** Off-centre Persons, and the second of two side-by-side Persons, lose coverage first.
- **If the Radar System's range is raised for the PoC room,** the 24° lens pays off: it gives 35 px at 7 m, against 28 px for the 32° and 18 px for the 50°.
- **The camera tilts to the torso band** rather than staying parallel to the Hanwha.
- **The lens is hard to swap once data collection starts,** because thermal training data is tied to it. Decide between 24° and 32° before then.
