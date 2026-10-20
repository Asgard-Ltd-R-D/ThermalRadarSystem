# Consuming the Radar System's ROS1 topics from another host (2026-10-20)

Question ([#3](https://github.com/Asgard-Ltd-R-D/ThermalRadarSystem/issues/3)): how can a Python process on a separate Linux host, or on macOS for development, consume pulse72's ROS1 Noetic topics, including the custom `arbe_msgs`?

Background: [pulse72 investigation](https://github.com/Asgard-Ltd-R-D/ThermalRadarSystem/blob/docs/radar-findings/docs/research/pulse72-radar-system.md). Paths written as `pulse72:realtime/...` are relative to `~/Documents/DeviceDetection` on pulse72. Paths written as `arbe_dev:/...` are inside the `arbe_dev` container. Everything on pulse72 was read-only. The Radar System was not running during this investigation, so nothing was observed live.

## Answer

**Use the `foxglove_bridge` that already runs on `ws://pulse72:8765`, with the Python packages `websockets` and `rosbags`.** This needs no changes on pulse72, and the same code runs on Linux and macOS. The bridge sends raw ROS1 bytes and puts the full message definition in its `advertise` message, so no `arbe_msgs` package is needed on the client.

**Native rospy cannot work off-host without changing how the Radar System is launched.** Because `ROS_HOSTNAME=localhost`, every socket binds to loopback only, the master included.

## The blocker: `ROS_HOSTNAME=localhost` binds everything to 127.0.0.1

- `headless_start.sh` hard-codes the setting inside the `docker exec ... bash -c "export ROS_HOSTNAME='localhost' && ... roslaunch ..."` string (`pulse72:realtime/headless_start.sh:143-150`). An operator cannot override it from the environment at launch time. Only `--ros-port` is configurable (`:20-35`).
- **Python nodes and the master:** `rosgraph.network.get_bind_address()` returns `127.0.0.1` when ROS_HOSTNAME is `localhost` (`arbe_dev:/opt/ros/noetic/lib/python3/dist-packages/rosgraph/network.py:241-262`, [upstream](https://github.com/ros/ros_comm/blob/noetic-devel/tools/rosgraph/src/rosgraph/network.py)). The XML-RPC server binds to that address (`rosgraph/xmlrpc.py:268`). This server is used by `rosmaster` and by every rospy node, so **port 11311 itself is unreachable from other hosts**.
- **C++ nodes** (Arbe drivers, `foxglove_bridge`): setting `ROS_HOSTNAME=localhost` sets `only_localhost_allowed_` ([transport.cpp](https://github.com/ros/ros_comm/blob/noetic-devel/clients/roscpp/src/libros/transport/transport.cpp)). TCPROS then listens on `INADDR_LOOPBACK` ([transport_tcp.cpp:382-394](https://github.com/ros/ros_comm/blob/noetic-devel/clients/roscpp/src/libros/transport/transport_tcp.cpp#L382-L394)).
- ROS1 requires "complete, bi-directional connectivity between all pairs of machines, on all ports" and that each machine "advertise itself by a name that all other machines can resolve". It describes `ROS_HOSTNAME=localhost` as the "Single machine configuration" ([ROS/NetworkSetup](http://wiki.ros.org/ROS/NetworkSetup), read via the [2024-12-28 archive](https://web.archive.org/web/20241228031754/http://wiki.ros.org/ROS/NetworkSetup) because wiki.ros.org blocks automated fetches).
- **What breaks for a remote rospy client:** it cannot reach the master at all (connection refused). Even with an SSH tunnel to 11311, the publisher URIs it gets back are `http://localhost:<ephemeral port>`, which point at the client's own machine.
  - Tunnelling every node's ephemeral XML-RPC and TCPROS port, plus a reverse tunnel for the master's `publisherUpdate` callback, is possible in principle. It is too brittle to use.
- **Fix:** set `ROS_IP=<pulse72 LAN IP>` (or a resolvable `ROS_HOSTNAME`) for the master and every node. Either edit `headless_start.sh:145`, which is a DeviceDetection change, or have the operator bypass the script and run `roslaunch` by hand, which forks the start procedure. Neither is acceptable under the rule of not modifying DeviceDetection.

## Options compared

| | A. Native rospy → pulse72 master | B. foxglove_bridge `:8765` (running) | C. rosbridge | D. Inside / beside `arbe_dev` (net=host) |
|---|---|---|---|---|
| Changes on pulse72 | Launch env must change (see above): **blocked** | **None** | Install `ros-noetic-rosbridge-server` into the container and run it separately | None to DeviceDetection. Adds our process to their host or container |
| Python libs (Linux) | rospy from ROS apt, or RoboStack | `websockets` + `rosbags` (pure Python + numpy) | `roslibpy` 2.1.0 | Container's rospy (Python 3.8, or 3.11 with `PYTHONPATH` as in `start_split.sh:97`) |
| Python libs (macOS) | RoboStack `robostack-noetic` (`ros-noetic-rospy` 1.17.4 py312, osx-arm64) | Same as Linux | Same as Linux | n/a (pulse72 only) |
| Custom `arbe_msgs` | Must build `arbe_msgs` on the client, or use `AnyMsg` + `rosbags` | Decode from the advertised schema with `rosbags` (verified, below) | rosbridge needs `arbe_msgs` on pulse72 (present); the client gets JSON dicts | Already generated: `arbe_dev:/ws/devel/lib/python3/dist-packages/arbe_msgs` |
| Pointclouds over the wire | Binary ROS1 | Binary ROS1 + 13 B header | JSON about 1.5× binary; CBOR about 1× | Loopback, no network |
| Reconnect after the GUI quits | Process restart needed | WebSocket retry loop | Retry loop, but rosbridge's own rospy node goes stale | Process restart needed |

### A. Native rospy

- Blocked, as described above.
- Even after the fix, two further things are needed:
  - pulse72's master must be able to call back into the client (`publisherUpdate`), so the client needs `ROS_IP` set and an open firewall on all ports (this matters on a macOS laptop).
  - The client needs Python classes for `arbe_msgs`.
- Packaging:
  - The PyPI-style `rospypi` index is unofficial and supports only Python 3.6-3.8 ([rospypi/simple](https://github.com/rospypi/simple)).
  - RoboStack's `robostack-noetic` channel lists linux-64, osx-64, osx-arm64, linux-aarch64 and win-64 ([RoboStack](https://robostack.github.io/GettingStarted.html); `ros-noetic-rospy-1.17.4-np2py312…` is in [prefix.dev/robostack-noetic](https://prefix.dev/channels/robostack-noetic) osx-arm64 repodata).

### B. foxglove_bridge (recommended)

- **What runs:** `ros-noetic-foxglove-bridge` 0.8.4 (`rosversion` in `arbe_dev`), included by `headless_3radars_cam.launch:288-291` with the stock launch defaults (`$(rospack find foxglove_bridge)/launch/foxglove_bridge.launch`):
  - address `0.0.0.0`
  - whitelists `.*`
  - `send_buffer_limit` 10 MB
  - capabilities `clientPublish, parameters, services, …`
  - no TLS
- **Maintenance:** the [repo](https://github.com/foxglove/ros-foxglove-bridge) is now ROS1-only, and its README says the ROS 1 package "is in maintenance mode and only bug fixes will be applied". The latest tag is 0.8.5, whose changes don't affect us ([CHANGELOG](https://github.com/foxglove/ros-foxglove-bridge/blob/0.8.5/CHANGELOG.rst)). The [ws-protocol spec](https://github.com/foxglove/ws-protocol/blob/main/docs/spec.md) is marked deprecated in favour of the Foxglove SDK. The bridge still speaks subprotocol `foxglove.websocket.v1` ([common.hpp:12](https://github.com/foxglove/ros-foxglove-bridge/blob/0.8.4/foxglove_bridge_base/include/foxglove_bridge/common.hpp#L12)).
- **Subscribe flow** ([spec](https://github.com/foxglove/ws-protocol/blob/main/docs/spec.md)):
  1. Connect with subprotocol `foxglove.websocket.v1`.
  2. The server sends `serverInfo`, then `advertise` with one channel per topic.
     - Each channel carries `encoding: "ros1"`, `schemaName: "arbe_msgs/arbePcFloatMsg"`, and `schema` set to the full concatenated `.msg` text. No `schemaEncoding` is sent ([nodelet 0.8.4 :45, :553-567](https://github.com/foxglove/ros-foxglove-bridge/blob/0.8.4/ros1_foxglove_bridge/src/ros1_foxglove_bridge_nodelet.cpp); [serialization.cpp:6-18](https://github.com/foxglove/ros-foxglove-bridge/blob/0.8.4/foxglove_bridge_base/src/serialization.cpp)).
     - The schema text comes from ros_babel_fish `computeFullText` (`arbe_dev:/opt/ros/noetic/include/ros_babel_fish/generation/description_provider.h:88`).
  3. The client sends `{"op":"subscribe","subscriptions":[{"id":N,"channelId":C}]}`.
  4. The server sends binary frames: `0x01 | uint32 LE subscription id | uint64 LE timestamp ns | raw ROS1 bytes`.
     - The bridge forwards `BabelFishMessage` buffers untouched, stamped with **receipt time** ([nodelet :844-850](https://github.com/foxglove/ros-foxglove-bridge/blob/0.8.4/ros1_foxglove_bridge/src/ros1_foxglove_bridge_nodelet.cpp#L844-L850); [websocket_server.hpp:993-1003](https://github.com/foxglove/ros-foxglove-bridge/blob/0.8.4/foxglove_bridge_base/include/foxglove_bridge/websocket_server.hpp#L993-L1003)).
     - For capture time, use `PcHeader.time` (radar hardware ms), not this timestamp.
- **Slow clients:** messages are dropped, not queued, once a client's send buffer reaches 10 MB ([websocket_server.hpp:968-974](https://github.com/foxglove/ros-foxglove-bridge/blob/0.8.4/foxglove_bridge_base/include/foxglove_bridge/websocket_server.hpp#L968-L974)). A consumer always sees fresh data, but must tolerate gaps.
- **Python libraries:**
  - `foxglove-websocket` (0.1.4) is a **server** library, "deprecated in favor of foxglove-sdk", with no client code ([PyPI](https://pypi.org/project/foxglove-websocket/); the wheel has only `server/`).
  - Use plain `websockets` ([PyPI](https://pypi.org/project/websockets/), Python ≥ 3.11 for 17.x) for the connection.
  - Use `rosbags` 0.11.9 ([PyPI](https://pypi.org/project/rosbags/), Apache-2.0, Python ≥ 3.10, [source](https://gitlab.com/ternaris/rosbags)) to decode. Its `get_types_from_msg` parses the `===…\nMSG: pkg/Type` concatenated format (`rosbags/typesys/msg.py:100,180,549-560`).
- **Custom message decoding, verified:**
  - One raw `/arbe/parsed/pointcloud_float/0` message and its genmsg full-text definition were taken from `pulse72:realtime/recordings/recording_20261019_134046.bag`.
  - It decoded on macOS with only `rosbags` (2049 points, all arrays the right length), and re-serialized byte-identically. Two gotchas:
    1. **`arbeFrameTypes` defines constants as `uint8 Frame_Type_3d_short = 1`.** rosbags only accepts UPPER_SNAKE constant names (`msg.py:103-108`), so it misreads these 8 constants as `uint8` fields and every later offset shifts. ROS1 never serializes constants, so strip constant lines before parsing (regex below).
    2. rosbags keys types as `arbe_msgs/msg/arbePcFloatMsg`, so insert `/msg/` before deserializing.
  - The bridge's schema text was not captured live. It comes from the same `.msg` files in the same `MSG:` format, so the same handling applies _(inferred)_.
- **Reconnect:**
  - Quitting the GUI runs `pkill -INT -f roslaunch` (`pulse72:realtime/gpu_stage.py:286-288`), which kills the bridge along with the rest of the ROS graph, so the socket closes.
  - The client retries `connect` with backoff and waits for `advertise`. The bridge polls the master at most every `max_update_ms` = 5000 ms ([README](https://github.com/foxglove/ros-foxglove-bridge)). Since 0.8.4 it also advertises topics that have no publishers ([CHANGELOG](https://github.com/foxglove/ros-foxglove-bridge/blob/0.8.5/CHANGELOG.rst)).
  - The client then re-subscribes **by topic name**. Never reuse channel ids across connections.
- **Exposure:** with the stock capabilities, anyone on the LAN can publish topics, set parameters and call services on the Radar System's graph, with no authentication. Reading needs none of these.

Minimal client sketch. It was run against a local stub server that uses the bridge's framing, with the real schema and message from the bag. It was not run against the live bridge, because the Radar System was down.

```python
import asyncio, json, re, struct, websockets
from rosbags.typesys import Stores, get_typestore, get_types_from_msg

CONST = re.compile(r"^[ \t]*\w+[ \t]+\w+[ \t]*=.*$", re.M)  # ROS1 constants aren't on the wire

async def stream(url, topics):
    async with websockets.connect(url, subprotocols=["foxglove.websocket.v1"], max_size=None) as ws:
        ts, subs = get_typestore(Stores.ROS1_NOETIC), {}
        async for frame in ws:
            if isinstance(frame, str):
                m = json.loads(frame)
                for ch in m.get("channels", []) if m["op"] == "advertise" else []:
                    if ch["topic"] in topics:
                        ts.register(get_types_from_msg(CONST.sub("", ch["schema"]), ch["schemaName"]))
                        subs[ch["id"]] = (ch["topic"], ch["schemaName"].replace("/", "/msg/", 1))
                        await ws.send(json.dumps({"op": "subscribe", "subscriptions": [{"id": ch["id"], "channelId": ch["id"]}]}))
            elif frame[0] == 0x01:
                sid, recv_ns = struct.unpack_from("<IQ", frame, 1)
                topic, typ = subs[sid]
                yield topic, recv_ns, ts.deserialize_ros1(memoryview(frame)[13:], typ)

async def main():
    while True:  # the ROS graph dies with the GUI; just reconnect
        try:
            async for topic, t, msg in stream("ws://pulse72.local:8765", {"/arbe/parsed/pointcloud_float/0"}):
                ...
        except (OSError, websockets.ConnectionClosed):
            await asyncio.sleep(2)
```

### C. rosbridge

- **Install cost:** rosbridge is not installed. It has to go into `arbe_dev` by apt (the image is Noetic and EOL, and the container has no apt lists cached), or it can be added on the side. It must be launched separately with the same loopback env, because `headless_start.sh` kills every ROS process at start (`:124-127`).
- **Wire format:**
  - The transport is JSON. `compression` can be `none`, `png`, `cbor` or `cbor-raw`. `cbor-raw` returns the raw message bytes ([ROSBRIDGE_PROTOCOL.md §3.4.4](https://github.com/RobotWebTools/rosbridge_suite/blob/ros1/ROSBRIDGE_PROTOCOL.md)).
  - JSON for one 2,247-point `arbePcFloatMsg` measured 147 KB against 97 KB binary (rosbridge-style dict → `json.dumps`, emulated on pulse72). The bins are integer-valued floats, so the ratio is only about 1.5×. The real cost is Python encoding in the rosbridge node at 60 messages/s.
  - `cbor-raw` would bring it to parity with option B, but then the client needs a `rosbags` decode anyway.
- **Reconnect:** `roslibpy` reconnects automatically (Twisted `ReconnectingClientFactory`, [comm_autobahn.py:13,60](https://github.com/gramaziokohler/roslibpy/blob/main/src/roslibpy/comm/comm_autobahn.py)). However, the rosbridge node is itself a rospy node and has the same stale-registration problem described under D.
- **Verdict:** it buys nothing over B, and it costs an install inside someone else's container.

### D. Inside or beside `arbe_dev`

- **Inside:** run `docker exec arbe_dev python3.11 consumer.py` with the same `PYTHONPATH` and `ROS_MASTER_URI=http://localhost:11311` that `start_split.sh:97-101` uses.
  - It gets native rospy and the generated `arbe_msgs` with nothing to install.
  - The start script's `pkill` pattern (`headless_start.sh:126`) would not kill it.
- **Beside:** run a sibling `ros:noetic` container with `--net=host` that builds `arbe_msgs` from the copy mounted at `/ws/src/arbe_phoenix_radar_driver/arbe_msgs`. This avoids touching their image.
- Both options put our process on pulse72, sharing CPU with a CPU stage that already drops frames. This is a question for #8, not a transport.
- **Reconnect:** rospy registers with the master in a retry loop at startup (`rospy/impl/registration.py:202-274`) and never re-registers after the master restarts. Calling `init_node` again does not help (`rospy/client.py:274`). The consumer must detect that the master has gone (for example, `rosgraph.Master.getPid()` fails or changes) and exit, so that a supervisor restarts it.

## Bandwidth

Measured from `pulse72:realtime/recordings/recording_20261019_134046.bag` (5.1 s, 103 messages per topic):

- **`arbePcFloatMsg`:** 82-90 KB on average per radar, up to 190 KB, about 43 B per point.
  - 3 radars × about 20 Hz × about 87 KB is about **5.2 MB/s (about 42 Mbit/s)**.
  - Through foxglove this is fine on wired gigabit Ethernet. It is marginal on Wi-Fi for macOS development, so subscribe to one radar there.
- **Camera:** the live topic `/hanwha_cam_0/image_raw` is raw `bgr8` (`hanwha_camera_node.py:145,281`) at 1920×1080 and about 20 fps.
  - That is about 6.2 MB per frame, or **about 124 MB/s (about 1 Gbit/s)**. It is not viable off-host, and it would hit the bridge's 10 MB send buffer and drop frames.
  - No compressed camera topic is published live. `/recording/image/compressed`, at about 300 KB per frame, exists only in the bags.
- **Suspect Region messages:** none are published today, because detections never leave `cpu_stage.py`. A message of under 1 KB at 20 Hz would be negligible on any of the four options.

## Recommendation

Consume over the existing foxglove_bridge with `websockets` + `rosbags`, using the reconnect loop above:

- It needs zero changes on pulse72 and no ROS on the client.
- The same code runs on Linux and macOS.
- It sends binary payloads.
- The schema arrives in-band.

Strip constant lines before calling `get_types_from_msg`. Take timing from `PcHeader.time`, not the frame timestamp. Pin to bridge 0.8.x and expect no new features, since the bridge is in maintenance mode.

Keep D (a consumer inside or beside `arbe_dev`) as the fallback if #8 places fusion on pulse72. Drop A unless the owners of DeviceDetection agree to set `ROS_IP` in `headless_start.sh`. Drop C.
