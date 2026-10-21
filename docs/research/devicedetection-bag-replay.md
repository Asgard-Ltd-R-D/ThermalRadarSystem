# Can DeviceDetection run headless on a recorded bag and emit Suspect Regions? (#2)

Read-only investigation of pulse72, 2026-10-20. Paths are relative to `~/Documents/DeviceDetection` (branch `pipeline` @ `9cc5288`). `realtime/radar_overlay.py` has uncommitted edits, so its line numbers are from the working tree; past line 1757 they are 3 lines higher than in the commit. **[V]** marks a fact verified in code or on disk. **[I]** marks an inference that nobody has run.

## Answer

- **Not as the Radar System ships today.** No unmodified script takes a bag, runs the live `Detection` chain headless, and writes the results to a file.
  - The headless bag runner (`profile_pipeline.py`) computes Detections and throws them away.
  - The paths that do write `<stem>_tracks.csv` need either a GUI with keypresses or the offline scoring chain.
- **Cheapest real Suspect Regions:**
  1. Use the 37 `_tracks.csv` files already on disk, which needs no compute.
  2. Or run one CPU-only command over a bag's existing `csv_cleaned` record. 139 bags have one.
- Both of these give per-frame windows from the *offline* chain with no track IDs. They become Suspect Regions only after re-tracking.
- A faithful headless run of the live chain needs a small driver script *outside* the repo that wraps `profile_pipeline.py`'s loop (see #7).

## Entry points that read a bag

| Entry point | Runs the live chain? | Headless? | Writes detections? |
|---|---|---|---|
| `start_split_sys1.sh` + `rosbag play` | yes, but CPU stage drops frames | no: `cpu_stage.py:153,214` uses `imshow` | **no**: split has no `TrackLogger` (`gpu_stage.py:100-106` toggles the bag recorder only) [V] |
| `radar_overlay.py` live serial (`start_overlay.sh`) | yes | no | **yes**, `<stem>_tracks.csv` on `b`/`B` with `a` on (`radar_overlay.py:2845-2863`, tap at `:2316-2323`) [V] |
| `radar_overlay.py --offline-only` / `start_offline.sh` | yes, after pressing `a` (default off, `:1895`, `:2799-2801`, `:2358`) | no: Tk file picker `:3666-3668`, OpenCV window | **no**: `TrackLogger` only exists when a live recorder does (`:3715-3742`) [V] |
| `bag_overlay_video.py` / `run_bag_overlay_video.sh` | **no**: no pose/seg/weapon engines wired (`bag_overlay_video.py:520-529`) | yes (mp4) | no [V] |
| `profile_pipeline.py` (repo root) | yes: pose → seg → optical kp → `match_pose_to_seg` → `WeaponDetectorEngine.detect` per frame, in order (`:219-267`) | **yes** | **no**: `detect()` return value discarded (`:259`) [V] |
| `models.evaluation.test_reproduce_notebook` | offline equivalent: `clean_frame` → `preprocess_frame` → XGBoost → hard NMS → gate (`:80-127`) | yes | **yes**, `<record>_tracks.csv` in `TrackLogger` columns (`:216-232`, `:295-296`) [V] |
| `models/realtime.py::run_example_video` | offline, on record folders | no (window by default) | stdout only; hard-coded `/home/itsik` paths (`:1049-1199`) [V] |

More facts about these entry points:
- `start_split.sh:74-76` and `start_overlay.sh:93-99` always start `headless_start.sh`. That script runs `pkill`s on roscore and friends, then launches the live radars and camera (`headless_start.sh:125-149`). So the start scripts **cannot be combined with `rosbag play` as-is**: live and bag data would mix. [V]
- On quit, `radar_overlay.py` and `gpu_stage.py` both run `pkill -INT -f roslaunch` (`radar_overlay.py:3932-3938`, `gpu_stage.py:286-288`). A replay inside the live container therefore kills a running Radar System. [V]
- The split really writes no tracks. `realtime/recordings/recording_20261019_134046.bag` (2026-10-19) has no `_tracks.csv` beside it. [V]
- `profile_pipeline.py` is stale:
  - Its default weapon model `models/outputs/models/xgboost_model_20260614T154932.json` is missing (`:150-151`).
  - Its default pose and seg models are `yolo11n-pose.pt` / `yolo26x-seg.pt` (`:152-153`), but live uses `yolo26n-pose` / `yolo26n-seg` at conf 0.6 (`radar_overlay.py:3618-3625`). [V]
  - Its `detect()` call still matches the current signature (`weapon_detector.py:679-692`). [V]

## Can `/recording/image/compressed` stand in for `/hanwha_cam_0/image_raw`?

- **Offline loaders: yes, natively.** The following all detect the recording layout and decode the JPEG themselves [V]:
  - `bag_overlay_video.load_frames_recording_layout` (`bag_overlay_video.py:296-340`)
  - the GUI offline mode (`radar_overlay.py:3094-3098`)
  - `profile_pipeline.py:194-197`
  - camrad extraction, through rosbags `AnyReader` (`legacy/pose_estimation/camrad_video_maker.py:39,1819-1831`)
- **ROS replay: not directly.** `RosReceiver` subscribes to `--camera-topic` as `sensor_msgs/Image` (`radar_overlay.py:1486`) and accepts only `bgr8` (`:166-180`). The bag holds `CompressedImage`, format `"jpeg"` (`:1834-1845`). A remap alone fails on message type [V].
  - Bridging it needs `rosrun image_transport republish compressed in:=/recording/image raw out:=/hanwha_cam_0/image_raw`.
  - [I] This should work because the image is built `FROM osrf/ros:noetic-desktop-full` (`realtime/Dockerfile:1`), which ships `image_transport_plugins`, and decoding a 3-channel JPEG yields `bgr8`.
- **Empty `frame_id`: irrelevant.** Nothing reads the camera `frame_id` (grep over `radar_overlay.py`, `bag_overlay_video.py`, `weapon_detector.py`, `gpu_stage.py`). [V]
- **JPEG quality 90** (`radar_overlay.py:1834`): the bag image is a re-encode, not the raw frame the live YOLO saw. Small Person and mask differences are possible. [I]
- **Restamping makes offline pairing exact.** `BagRecorder` writes each already-paired frame like this:
  - The radar `RosHeader.stamp` is overwritten with the camera stamp S (`:1817`).
  - The image header is S.
  - The image's bag time is S + 1 ns (`:1844`).
  - The offline loader buckets radar and image by equal header stamps (`bag_overlay_video.py:315-320`), so it reproduces the pairing the Radar System made at record time. [V]
- **Restamping doesn't matter for `rosbag play`.** The receiver pairs on the untouched `PcHeader.time` (radar hardware ms) against the image header (`:1531`, `:1559-1561`). [I] Replay should re-pick the same radar frames, since live `--cam-delay` defaults to 0 (`:3535`).
- **Join keys between files:**
  - camrad names record files by bag time (`camrad_video_maker.py:1831,2365`). [V] So `csv_cleaned` timestamps should be S + 1 ns. [I]
  - `TrackLogger` keys on `rospy.Time.from_sec(stamp).to_nsec()`, a float round trip (`:2320`).
  - Join tracks to frames by nearest timestamp, not by equality. [I]

## What already exists, and the commands

### A. Zero compute: tracks files already on pulse72

- `models/outputs/realtime_eval_repro/` holds 82 `<record>_tracks.csv` files. They are untracked in git and dated 2026-07-14.
- They were written by `test_reproduce_notebook` over `realtime/csv_cleaned` test records.
- **37 of them match a bag** in `recordings_14_05` or `recordings_27_05`. [V]
- What the rows contain [V]:
  - Rows are post-NMS windows with `prob ≥ 0.85`.
  - `cx, cy, w, h` are raw fisheye pixels. `height_score` is filled in.
  - `person_track_id` and `weapon_track_id` are `-1`, and `mature` is `0`, because the offline chain has no tracker (`test_reproduce_notebook.py:225`).
  - Frames with no detection get a sentinel row.
- Which model produced them is not recorded. [I] Probably `round2_winner.json` (`realtime/TRACK_LOGGING.md:170-173`), not the deployed `exp6_winner_tuned.json`.

### B. One CPU-only command, for any bag with a `csv_cleaned` record (139 bags)

Run it on the host, with nothing written inside the repo:

```bash
mkdir -p /tmp/one && ln -s ~/Documents/DeviceDetection/realtime/csv_cleaned/<stem> /tmp/one/
cd ~/Documents/DeviceDetection && python3 -m models.evaluation.test_reproduce_notebook \
  --records-dir /tmp/one --split /nonexistent \
  --model models/outputs/exp6_winner_tuned.json --work-dir /tmp/out
# → /tmp/out/<stem>_tracks.csv
```

- **Needs:** no GPU, TensorRT, ROS or container.
  - The host's python3.10 has xgboost 3.2, pandas, numba, scikit-learn and opencv. [V] Running on the host is [I]. The script says to run it "in the same env that runs the pipeline" (`test_reproduce_notebook.py:35-37`).
  - In the container, run from `/` with `--model /models/outputs/exp6_winner_tuned.json`.
- `--split /nonexistent` selects every record directory (`:235-242`).
- The default gate is `nms_hard_max_score_thresh = 0.85` (`:256`, `models/config.py:108`).
- The CSV is written before the script's later evaluation and asserts (`:295-296`).
- **It differs from the live chain** [V]:
  - Its keypoints and masks come from camrad extraction with `yolo11n-pose.pt` + `yolo26x-seg.pt` (`camrad_video_maker.py:1022-1023`, `models/data_maker/auto_data.py:82-95`).
  - It scores whole frames, not per Person (`weapon_detector.py:331,1011`).
  - It has no ByteTrack Person ID and no online HeightTracker.
- **To get Suspect Regions:** re-track the rows with `models/evaluation/exp6_metrics.py::assign_tracks` (`:109`, the notebook rule). A Suspect Region is an alarm once its age reaches `tracker_min_age = 5` (`models/config.py:137`).
- **Bags without a record** go through `models/data_maker/auto_data.py::save_data`.
  - It has no CLI; its `__main__` hard-codes `/home/itsik` paths (`:138-150`).
  - It needs YOLO `.pt` files and the launch config or intrinsics resolvable from the bag folder (`camrad_video_maker.py:1442-1454`). The `recordings_*` folders hold only `.bag` and `.csv` files. [V] Whether extraction then works is [I].

### C. Live chain, unmodified, with a GUI (real `Detection`s with Person and region track IDs)

```bash
# host, live Radar System stopped (quitting pkills roslaunch); only realtime/ and models/ are mounted
cp ~/Documents/recordings_14_05/<stem>.bag ~/Documents/DeviceDetection/realtime/
xhost +SI:localuser:root
docker exec -it -e DISPLAY=:0 -e QT_X11_NO_MITSHM=1 arbe_dev bash -lc '
  source /opt/ros/noetic/setup.bash && cd /ws && source devel/setup.bash &&
  export PYTHONPATH=/ws/devel/lib/python3/dist-packages:/opt/ros/noetic/lib/python3/dist-packages:$PYTHONPATH &&
  (roscore &) ; sleep 3 ;
  (rosrun image_transport republish compressed in:=/recording/image raw out:=/hanwha_cam_0/image_raw &) ;
  python3.11 /ws/radar_overlay.py --config /ws/launch_config_sys1.yaml \
    --weapon-model /models/outputs/exp6_winner_tuned.json'
# in the window: press a, then B (video-only record + tracks log)
# second shell:  docker exec arbe_dev bash -lc 'source /opt/ros/noetic/setup.bash && rosbag play -r 0.25 /ws/<stem>.bag'
# when the bag ends: press B, skip labeling, q  →  realtime/recordings/recording_<ts>_tracks.csv (+ _config.json)
```

- **Needs** [V]:
  - the `arbe_dev` container (`realtime/lane.env`), which has ROS Noetic, Python 3.11 and TensorRT
  - a GPU (`--gpus all`, `docker_setup.sh:33`)
  - the TRT engines `realtime/yolo26n-{pose,seg}.engine`, which `PoseEngine` and `SegEngine` prefer over the `.pt` files (`radar_overlay.py:572-573,895-896`)
  - an X display
- The command assembly is [I]; nobody has run it.
- **Slow the replay down.** The receiver queue holds 3 frames and drops the oldest (`:1692-1698`). `-r 0.25` is a knob to tune until frames stop dropping.

### D. Headless live chain via an external driver (recommended for #7)

- A ~30-line script outside the repo would:
  1. import `profile_pipeline.py`'s pieces;
  2. repeat its loop with the live models (`realtime/yolo26n-pose.pt`, `yolo26n-seg.pt`, `exp6_winner_tuned.json`);
  3. pass each `detect()` result to `realtime/track_logger.py::TrackLogger` (`:55-122`).
- That gives `<stem>_tracks.csv` with every frame and no GUI.
- It needs what C needs, except ROS and the display. [I] Not run.
- [I] On the host there is no `tensorrt`, so the `.engine` files sitting next to the `.pt` files would fail to load. Use the container, or point at copies of the `.pt` files that have no `.engine` beside them.
