# ThermalRadarSystem

Thermal + radar concealed weapon detection for the Meyuhadim project.

## Architecture

- **Radar** stays on pulse72, in the DeviceDetection repo (ROS1 Noetic in Docker).
  The Arbe radar point cloud is published on `/pcl_combined_layer`.
- **Thermal + fusion** run on this side and subscribe to the radar over the network:

  ```sh
  export ROS_MASTER_URI=http://pulse72.local:11311
  export ROS_IP=<this machine's IP>
  ```

- Keep both machines' clocks synced (e.g. chrony) so radar and thermal timestamps match.
- Offline development: `rosbag play` the recordings from DeviceDetection
  (`realtime/14_05`, `realtime/27_05`, `realtime/jun29`).

**Open decision:** is thermal a separate detector whose results are fused with
pulse72's per-person weapon scores, or an extra input to the XGBoost weapon model?
The first needs a small ROS publisher on pulse72; the second needs the raw radar
points per person.

## Docs

- `docs/meyuhadim/`: pipeline notes and customer meeting questions.
