# ThermalRadarSystem

Thermal + radar concealed-object detection. Vocabulary in `GLOSSARY.md`; decisions in `docs/adr/`.

- **Radar System** runs on pulse72 (`~/Documents/DeviceDetection`, branch `pipeline`) inside the `arbe_dev` container (ROS1 Noetic), started by `~/run.sh`. It publishes raw radar and camera topics but **no detection topic**; details in `docs/research/pulse72-radar-system.md`.
- **Connect:**
  - **From this machine,** only SSH works: `ssh pulse72.local` goes over IPv6 link-local.
  - **IPv4 from the office LAN fails,** because pulse72 routes replies to 192.168.1.0/24 out its wired NIC. So `foxglove_bridge` (IPv4 `0.0.0.0:8765`) can't be reached from here.
  - **A host on the rig's wired sensor network** uses `ws://10.20.30.60:8765`, and only while the pipeline runs. See "Network" in `docs/research/pulse72-radar-system.md`.
  - **Native rospy doesn't work off-host** (`ROS_HOSTNAME=localhost`).
- **Clock sync:** keep both machines synced (e.g. chrony) so timestamps match.
- **Offline:** bags are in `~/Documents/recordings_{11_05,14_05,27_05}` on pulse72 (raw radar + JPEG, no detections).
- **Thermal** verifies the Radar System's Suspect Regions; it does not detect on its own (`docs/adr/0001-thermal-verifies-suspect-regions.md`).
