# We run our own copy of the Radar System's detection chain

The Radar System computes Suspect Regions but publishes none: there is no topic, log or file (`docs/research/pulse72-radar-system.md`). We may not modify DeviceDetection, and for now we can't ask the radar team for anything. So we run a read-only copy of its detection chain (code, models and config), the **replica**, as its own process on our host. That host joins the rig's wired sensor network. It takes the camera straight from the Hanwha over RTSP and the radar from the existing foxglove_bridge at pulse72's wired address (`ws://10.20.30.60:8765`), and it writes one JSON line per processed frame to fusion. Replayed over a recorded bag, the same replica produces the walking skeleton's Suspect Regions.

## Considered options

- **The radar team publishes a topic.** This is the cleanest option, but asking is off the table for now. If that changes, their topic replaces the replica, and the payload can stay the same.
- **A hook that launches their unchanged `cpu_stage` under our wrapper.** Its output would match theirs exactly. But their rig would then start through our launcher, and the hook patches their private functions.
- **The replica on pulse72, beside their container.** This needs no new cable. But our process would live permanently on their rig, competing with a CPU stage that already drops frames. Thermal-to-region pairing would also cross hosts.
- **A replay-only v1.** This leaves the spec with no live source.

## Consequences

- **Compute runs twice.** Our host needs a GPU for YOLO pose and segmentation at 20 fps.
- **Our Person and region track IDs differ** from the ones on the Radar System's red boxes. Its alarm and our Alerts can disagree.
- **Our copy goes stale** when the radar team changes code, models or config. Each JSON Lines stream opens with a `source` line that records which copy produced it.
- **Live angle nudges in their GUI never reach our copy.** They are never saved to the YAML either.
- **One host clock stamps both the RGB and the thermal frames, but neither stamp is the capture time.** A shared clock removes clock offset, not pipeline delay.
  - **RGB latency:** the RGB stamp is taken after the camera's encode, RTSP transport and decode. We measure that latency once on the rig, for example by filming a millisecond clock on our host's screen.
  - **Radar↔RGB pairing:** radar `PcHeader.time` is the capture time.
    - The replica sets their existing `--cam-delay` knob ("camera RTSP latency"; it pairs each frame with the radar snapshot that many seconds earlier) to the measured RGB latency. Each RGB frame then pairs with the radar snapshot captured with it.
    - Their live pipeline runs with `--cam-delay` 0, so its pairing can differ from ours by its own decode latency. That's one more accepted replica difference.
    - NTP keeps our clock in step with pulse72's, and their own pairing already relies on pulse72's clock matching the radar timebase.
  - **RGB↔thermal pairing:** the thermal stamp is taken after USB acquisition. The RGB↔thermal latency difference is applied when fusion pairs a region with a thermal frame, and the remaining jitter has to fit inside the 25 ms pairing budget.
- **The wire carries the fisheye box and `depth_m`.** Fusion back-projects the region itself, so the replica and fusion must use the same calibration (`Hanwha.cam` and the RGB↔car entries of the YAML).
  - That calibration is identified once per stream, not on every frame: the opening `source` line carries a hash of the calibration the replica loaded.
  - Fusion refuses a stream whose hash differs from the calibration it loaded, so a restart after a calibration change can't silently shift the thermal region.
