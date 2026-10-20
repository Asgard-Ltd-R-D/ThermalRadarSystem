# ThermalRadarSystem

Thermal + radar concealed-object detection. Vocabulary in `GLOSSARY.md`; decisions in `docs/adr/`.

- **Radar System** runs on pulse72 (`~/Documents/DeviceDetection`, branch `pipeline`) inside the `arbe_dev` container (ROS1 Noetic), started by `~/run.sh`. It publishes raw radar and camera topics but **no detection topic**; details in `docs/research/pulse72-radar-system.md`.
- **Connect** from this machine: `foxglove_bridge` on `ws://pulse72.local:8765`, or `ROS_MASTER_URI=http://pulse72.local:11311` (the launch sets `ROS_HOSTNAME=localhost`, so remote rospy likely needs that fixed). Either only works while the pipeline runs.
- **Clock sync:** keep both machines synced (e.g. chrony) so timestamps match.
- **Offline:** bags are in `~/Documents/recordings_{11_05,14_05,27_05}` on pulse72 (raw radar + JPEG, no detections).
- **Thermal** verifies the Radar System's Suspect Regions; it does not detect on its own (`docs/adr/0001-thermal-verifies-suspect-regions.md`).
