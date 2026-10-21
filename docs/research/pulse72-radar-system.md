# The Radar System on pulse72 (read-only investigation, 2026-10-20)

Gathered over SSH without changing anything on pulse72. Paths are relative to `~/Documents/DeviceDetection` unless noted otherwise. Claims marked _(inferred)_ were not verified.

## What runs

- The code that runs is **`~/Documents/DeviceDetection`**, branch `pipeline` @ `9cc5288` (`git@github.com:Axon-Pulse/DeviceDetection.git`). It has uncommitted edits to `realtime/launch_config_sys1.yaml` (radar 2 IP/ports, pitch).
  - `~/run.sh` runs `realtime/start_split_sys1.sh --pipeline`.
  - `realtime/lane.env` selects the Docker container `arbe_dev`, which mounts `realtime/` → `/ws` and `models/` → `/models`, with `net=host`.
  - The system is started by hand; there is no autostart.
- The other copies:
  - `~/Documents/SPECOPS/DeviceDetection` is stale (branch `test`).
  - TESTING and DEMONSTRATION copies no longer exist.
  - `~/Documents/Asgard` is a no-remote import of `pipeline @ 9cc5288`, made 2026-10-20.
- **Host:** Ubuntu 22.04, 20 cores, 30 GB RAM, NVIDIA RTX PRO 2000 (8 GB, CUDA 13.2). ROS is not installed on the host.
- **Container:** Ubuntu 20.04, **ROS1 Noetic**. The app runs on Python 3.11 with torch 2.11, ultralytics 8.4.70, TensorRT 11.1, xgboost 3.2, opencv 4.11.

## Sensors and geometry

- **3× Arbe Phoenix 4D radars**, `range_mode: VeryShort` (`realtime/launch_config_sys1.yaml:84-109`):
  - Front: x 0, yaw −2.5°.
  - Right: x +0.5 m, yaw −19.5°.
  - Left: x −0.5 m, yaw +11.5°.
  - All have pitch 2° and z 0.
- Radar coverage in the bags: 0.3–10.2 m, azimuth about ±52°, elevation about ±18°, 0.0625 m range resolution.
- **1× Hanwha SLA-T2480V RGB fisheye camera**, 1920×1080 at about 20 fps:
  - Intrinsics fx = fy = 930, cx 960, cy 540, k1–k4 fisheye.
  - Offset from the front radar is [−0.072, 0.1, 0] m (`CameraCalibrationFiles/Hanwha.cam`).
- Frames: radar and "car" frames are x = right, y = forward, z = up, in metres; the car origin is the front radar antenna (`radar_overlay.py:344,1667`). Mounting height above the floor is undocumented.
- **RGB↔car angles can change at runtime.** Keys `1/4 2/5 3/6` nudge `cam_alpha/beta/gamma` by ±1°. In the split path, `gpu_stage` rebuilds R and t every frame from the nudged values (`radar_overlay.py:2157,2735-2746`). The nudges are never saved: the `w` key writes only the radar install values (`:2934-2948`).
- **No thermal camera exists**, and nothing in the repo mentions thermal.

## Network (checked 2026-10-21)

- **The rig's sensors sit on a private wired network.**
  - The radars (10.20.30.40, .45, .46; `launch_config_sys1.yaml:111-114`) and the Hanwha (192.168.1.101; `:140`) share pulse72's one wired NIC, `enp134s0`, which holds 10.20.30.60/24 and 192.168.1.250/24.
  - That network has no gateway, and nothing on the office LAN can reach it.
- **pulse72 reaches the office LAN over Wi-Fi** (`wlp132s0f0`, 192.168.1.152/24). `pulse72.local` resolves to that address and to an IPv6 link-local address on the same interface.
- **Both networks use 192.168.1.0/24, which breaks IPv4 from the office LAN.**
  - pulse72 has no source-based routing (`ip rule` shows only the defaults) and prefers the wired /24 route (metric 100 vs 600).
  - So its replies to any office host go out the wired NIC: `ip route get 192.168.1.55` gives `dev enp134s0 src 192.168.1.250`.
  - From the dev Mac, a TCP connect to 192.168.1.152:22 fails. `ssh pulse72.local` works only because it goes over IPv6 link-local (`fe80::…%wlp132s0f0`).
  - Fixing this would need source-based routing on pulse72, which is a change to their host.
- **foxglove_bridge binds `0.0.0.0:8765`**, which is IPv4 only _(inferred: not checked while it was running)_.
  - **From the office LAN:** unreachable as pulse72 is configured, because pulse72 routes its IPv4 replies out the wired NIC.
  - **From a host on the wired network:** use `ws://10.20.30.60:8765`. 10.20.30.0/24 doesn't overlap the office LAN, but `pulse72.local` (192.168.1.152) and 192.168.1.250 both sit in the shared 192.168.1.0/24, so a host on both networks would mis-route to them.
- **The camera stream:** the node pulls `rtsp://192.168.1.101:554/1/media.smp` as H.264, with RTP interleaved over TCP and no credentials (`headless_3radars_cam.launch:273`; `hanwha_camera_node.py:162-173`).
  - **From a host on the wired network:** give its wired NIC a free 192.168.1.x address alongside its 10.20.30.x one, as pulse72 does with .250.
    - The camera has no gateway, so it can only reply to an address in its own subnet.
    - Check that the address is free first, for example with `arping`.
  - **If that host is also on the office LAN:** route the camera through the wired interface explicitly, e.g. `ip route add 192.168.1.101/32 dev <wired-if> src <wired-192.168.1.x>`.
    - Longest-prefix match makes that /32 win over both 192.168.1.0/24 routes.
    - Without it, RTSP can leave over office Wi-Fi and fail, or reach an unrelated host.
  - The SLA-T2480V is only a lens module. Hanwha lists it as a lens for the XNB-6001 body, which serves up to 20 unicast RTSP clients. That this rig's body is an XNB-6001 is _(inferred)_.
- **Clock:** pulse72 keeps time with chrony, NTP-synchronized.

## Pipeline

1. `roslaunch headless_3radars_cam.launch` starts the ROS master and these nodes:
   - 3× Arbe receivers, publishing `/arbe/parsed/pointcloud_float/{0,1,2}` (`arbe_msgs/arbePcFloatMsg`, about 20 Hz each).
   - DSP and visualization nodes.
   - `hanwha_camera_node.py`, publishing `/hanwha_cam_0/image_raw`.
   - A sync node.
   - `foxglove_bridge` on `ws://0.0.0.0:8765`, with all topics whitelisted.
2. `gpu_stage.py` (ROS node `wdet_gpu_stage`) does the following:
   - Converts radar bins to xyz (`radar_overlay.py:1595-1672`).
   - Pairs each camera frame with the nearest radar snapshot (`:1547-1582`).
   - Runs YOLO26n pose with ByteTrack Person IDs and YOLO26n seg, both TensorRT at conf 0.6 (`:3608-3626`).
   - Hands frames to `cpu_stage` over `/dev/shm` and a UNIX socket (`gpu_stage.py:245-259`).
3. `cpu_stage.py` (no ROS) runs `WeaponDetectorEngine.detect` (`radar_overlay.py:2281`). For each Person:
   - Radar points are transformed to the car frame, gated at y ≤ 5 m, projected into the image and kept if they fall inside the Person's mask (`weapon_detector.py:736`).
   - A GMM clutter filter runs, plus a 0.5–3 m median range gate (`models/config.py:10-11`).
     - A Person outside the gate gets **no Suspect Regions** (`models/realtime.py:456,770`).
     - Line 770 appears to let `"out_of_fov"` through to classification. The status actually carries a suffix (`"out_of_fov: median_y=…"`), so that exact-match check never succeeds.
     - The classifier's training data is cleaned to the same range (`models/pipeline/clean.py`).
   - Sliding windows run over the torso.
   - XGBoost gives P(weapon) per window, followed by hard NMS (IoU 0.1, gate **0.85**; `models/config.py:107-108`).
   - A per-Person height tracker assigns region track IDs.
4. The Radar System's **"alarm"** is a region track that is ≥ 5 frames old (`models/config.py:137`, `weapon_detector.py:1321`). It is drawn as a red box in an OpenCV window.

## What a detection carries

- `Detection` (`weapon_detector.py:231-245`) has these fields:
  - `track_id` (Person)
  - `weapon_track_id`
  - `cx, cy, w, h` in **raw distorted fisheye pixels**
  - `prob`
  - `weapon_track_age`
  - `height_score`
- It has **no 3D position or range**, but the radar points behind each window do have 3D coordinates.
- The Person bbox is computed (`pose_people`) but is not in `Detection`.
- The tunables live in one file, `models/config.py:79-153`. Restart to change them.

## Outputs and interfaces

- **No detection topic or message exists.** Detections live only inside `cpu_stage.py`.
  - `TrackLogger` CSV (`realtime/track_logger.py`) is created only in the serial path (`radar_overlay.py:3742`), so the split launcher writes no tracks file.
  - **No log carries detections either.**
    - `cpu_stage` writes no log; its stdout goes to the `docker exec -it` terminal (`start_split.sh:104-118`).
    - The `gpu_stage` logs hold only sync, model-load and frame-drop lines.
    - `--weapon-debug` prints per-Person `median_y` and counts, but no boxes, probabilities or IDs (`weapon_detector.py:773,1205,1293`).
- `/pcl_combined_layer` is an unused config entry left over from the Arbe GUI. Nothing publishes it.
- The ROS graph exists only while the GUI runs; quitting kills roslaunch (`gpu_stage.py:286-288`).
- `headless_start.sh:145` sets `ROS_HOSTNAME=localhost`, so remote rospy subscribers will likely fail _(inferred)_.
- `rosbridge_server` is not installed; `vision_msgs` is.

## Timestamps and rates

- **Radar:**
  - `RosHeader.stamp` is set from host `ros::Time::now()` at publish (`pointcloud_node.cpp:2363`).
  - `PcHeader.time` is radar hardware capture time in Unix ms, and is what the app buffers on (`radar_overlay.py:1531`).
- **Camera:** the stamp is pulse72's `rospy.Time.now()` at the moment the decoded frame is pulled, so it includes the network and decode delay (`hanwha_camera_node.py:262-274,282`). An RTCP mapper exists (`:86-108`) but is never called. _(Corrected 2026-10-21: this line used to say "RTCP sender report plus RTP delta".)_
- **Radar↔camera pairing:** each camera frame gets the radar snapshot nearest `cam_stamp − cam_delay` (`radar_overlay.py:1406-1408,1560`).
  - `--cam-delay` ("camera RTSP latency") defaults to 0 (`:3535`), and the split launcher doesn't pass it.
  - The `,` and `.` keys nudge it live by 50 ms (`:2753-2758`). It is never saved.
- **Detections:** a detection frame carries the camera stamp (`radar_overlay.py:1582`).
- **Rates:**
  - Radar runs at about 20 Hz × 3 and the camera at about 20 fps.
  - The GPU stage can do about 50 fps.
  - The CPU stage falls behind and drops frames (`realtime/logs/gpu_stage-20261019-133846.log`).

## Recordings

| Location | Bags | Size | Total duration | Labels (bags) |
|---|---|---|---|---|
| `~/Documents/recordings_11_05` | 104 | 10.2 GB | 14.1 min | pistol 48, ied 31, none 24, no sidecar 1 (`calib.bag`) |
| `~/Documents/recordings_14_05` | 158 | 10.8 GB | 15.2 min | pistol 112, none 44, no sidecar 2 (`recording_*.bag`) |
| `~/Documents/recordings_27_05` | 91 | 9.5 GB | 14.0 min | pistol 59, ied 24, none 2, no sidecar 6 (`recording_*.bag`) |

- Every bag holds `/arbe/parsed/pointcloud_float/{0,1,2}` and `/recording/image/compressed`, 5–18 s each. **The bags contain no detections.**
- Each bag except the "no sidecar" ones has a `.csv` sidecar with `initials, weapon, visibility, direction, clothing`. The label counts above come from the sidecars' `weapon` column.
- `recordings_11_05/metadata.csv` indexes 101 of its bags and disagrees with the sidecars on 14 of them: 10 `axon` bags are `toy_pistol` there but `pistol` in the sidecar, 2 `_none_` bags are `ied` there, and the 2 `_MP` bags are `pistol+ied` with two people there but `pistol` in the sidecar.
- Per-frame weapon boxes are in `realtime/csv_cleaned/<record>/bbox_<ts>.csv`.
- `realtime/{14_05,27_05,jun29}` are extracted CSV/PNG, not bags. No `jun29` bags exist.
