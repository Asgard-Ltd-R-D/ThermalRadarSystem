# ThermalRadarSystem

Thermal + radar weapon detection .

- **Radar** runs on pulse72 (DeviceDetection repo) and publishes on `/pcl_combined_layer`.
- **Connect** from this machine:
  ```sh
  export ROS_MASTER_URI=http://pulse72.local:11311
  ```
- **Clock sync:** keep both machines synced (e.g. chrony) so timestamps match.
- **Offline:** `rosbag play` the DeviceDetection recordings (`realtime/14_05`, `27_05`, `jun29`).
- **Open:** thermal as a separate detector, or as an extra input to the weapon model?
