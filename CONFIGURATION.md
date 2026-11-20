# Configuration reference

Every parameter of the Mistral drone stack that is set outside the node code, where it lives, what
it does and whether it has been verified on this drone. Read this before changing a value; see
`INSTALL.md` for installing and running, `RELEASE.md` for cutting a release.

## 1. How configuration is layered

Values are applied in this order, later layers winning:

| layer | file | set when | changes need |
|---|---|---|---|
| node defaults | sima-core, per package | compile time | a sima-core change |
| per-stack common config | `src/mistral_bringup/config/<stack>/<stack>.yaml` | build time, shipped in the release | a rebuild, or an edit of the installed copy on the board |
| per-mode half | `<stack>.hw.yaml` or `<stack>.sim.yaml`, picked by `mode:=` | launch | same |
| board-resolved | `config/camera/camera.detected.yaml` | written by `deploy.sh` on install | a redeploy |
| launch arguments | `robot.launch.py` | each launch | editing the run wrapper, or a manual `ros2 launch` |
| deploy-time knobs | `deploy.yaml` (repo) → `deploy.env` (release) → `deploy.sh` flags | install | a redeploy |

On the board the config tree is installed under
`<remote_dir>/install/share/mistral_bringup/config/`, by default `/root/sima_ws/install/share/
mistral_bringup/config/`. Editing a file there and restarting the stack is the quickest way to try
a value; the change lasts until the next `deploy.sh`, which replaces the whole `install/` tree.
Anything meant to stick goes into this repo and ships with the next release.

Model archives are **not** configuration: they are provisioned from `models/models.manifest.yaml`
into the install share, and a config `model_path` selects which one a node loads (section 9).

## 2. Deploy-time knobs

### `deploy.yaml` (repo) and `deploy.env` (release)

`gen_release.sh` freezes these from `deploy.yaml` into the release's `deploy.env`; `deploy.sh` and
`provision.sh` read `deploy.env`.

| key | default | what |
|---|---|---|
| `board` | `root@10.42.0.130` | development default target; `deploy.sh --board` overrides |
| `remote_dir` | `/root/sima_ws` | where the overlay is installed; `deploy.sh --remote-dir` overrides |
| `ros_domain_id` | `25` | fleet DDS domain. Must differ from the rover's 24. `deploy.sh --domain` overrides per install |
| `bringup` | `mistral_bringup` | the package whose `package.xml` selects the capabilities built |
| `sys_config` | `mistral_config.yaml` | the system config passed to every launch include |
| `dds_profile` | `config/dds/fastdds_large.xml` | Fast DDS profile for large (image) messages, exported as `FASTRTPS_DEFAULT_PROFILES_FILE` by the run wrapper |
| `bundle_libs` | `[]` | native libraries to ship when the board's ABI lacks them |
| `runtime_pip` | the vendored GTSAM wheel | wheels installed on the board at deploy |
| `vo_pip` | the vendored OpenCV 4.10 wheel | wheels for the VO process alone, in `install/lib/vo_pydeps`. eLxr 2.x only (VP-16801) |
| `neat_stage` | `vendor/neat/neat-stage-0.4.0-yolox-seg-pose.tar.gz` | SiMa's offline NEAT stage, shipped as `prereq/neat/`; `deploy.sh` installs it on a board with another NEAT (VP-16803) |
| `models` | `models/models.manifest.yaml` | the model manifest to provision |

### `deploy.sh` flags

| flag | effect |
|---|---|
| `--mode hw\|sim` | hardware (default) or simulation. Selects the `.hw.yaml`/`.sim.yaml` halves and, in sim, turns the camera and radar stacks off |
| `--no-camera` | declares no camera. Skips the camera probe, bakes `camera:=false` into the wrapper, gate profile `replay` |
| `--no-radar` | declares no radar. Bakes `radar_driver:=false`, the TUI's radar row in STANDBY, gate profile `hw_no_radar` |
| `--radar` | declares a radar where the system config's `sensors.radar.enabled` is `false` (the x650's). Without either flag that key decides |
| `--no-perception` | leaves the perception stack out (`perception:=false`); allowed in sim |
| `--domain N` | ROS_DOMAIN_ID for this install (0–101) |
| `--isolated` | `ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST`: the board talks only to itself. Not with sim |
| `--sim-host IP` | sim only: exported as `ROS_STATIC_PEERS` so the board finds the simulator |
| `--remote-dir DIR` | install somewhere other than `remote_dir` |
| `--keep-neat` | leave the board's NEAT as it is even when it differs from the release's; warns instead of installing `prereq/neat/` |
| `--sys-config FILE` | the system config in `config/system/` the wrappers launch with, default `sys_config` from `deploy.yaml`: `mistral_config.yaml` (MIPI camera) or `x650_config.yaml` (the x650's USB camera) |
| `--run` | launch after installing |

`provision.sh` takes `--mode`, `--no-camera`, `--no-radar` and additionally `--no-fc` (no flight
controller: reported `n/a` instead of `MISSING`).

### Environment variables read by the scripts

| variable | read by | what |
|---|---|---|
| `SIM_IMAGE` | `provision.sh` | the simulator container image to check for, default `mistral-sim:6.1` |
| `BRIDGE_WS` | `provision.sh`, `run_bridge.sh` | a hand-built Foxglove bridge workspace, default `/nvme/foxglove_ws/install` |
| `HOTSPOT_PASSWORD` | `provisioning/wifi/Assets/*.sh` | the board hotspot passphrase; the scripts refuse to run without it |
| `ROS_DOMAIN_ID` | `setup_overlay.sh` | inherited if set; the wrappers export the deploy value first |

## 3. Launch arguments (`robot.launch.py`)

The run wrapper launches `ros2 launch mistral_bringup robot.launch.py sys_config:=<sys_config>
mode:=<mode>` plus whatever the deploy flags baked in. Available arguments:

| argument | default | what |
|---|---|---|
| `sys_config` | `mistral_config.yaml` | system config file in `config/system/` |
| `mode` | `hw` | `hw` or `sim`; selects the per-mode config halves |
| `camera` | `true` in hw, `false` in sim and where `sensors.camera.enabled` is `false` | start the camera publisher |
| `radar_driver` | `true` in hw, `false` in sim and where `sensors.radar.enabled` is `false` (the x650) | start the TI radar driver |
| `perception` | `true` | start segmentation + radar fusion |
| `vo` | `true` | start visual odometry |

A failing include aborts the whole launch, so a stack that cannot come up must be switched off
here rather than left to fail.

## 4. System config: `config/system/mistral_config.yaml`

Loaded by sima-core's `system_config`; the keystone every include receives as `sys_config`.

| key | value | status |
|---|---|---|
| `system.robot_name` | `mistral_drone` | |
| `system.use_sim_time` | `false` | |
| `system.description_package` | `mistral_description` | package does not exist yet; its `exec_depend` is commented out |
| `sensors.camera.enabled` / `.type` | `true` / `mipi_csi` | the driver `launch_camera.launch.py` starts, that camera's parameters and mount on the drone (`config/camera/<type>.yaml`) and the VO parameters `launch_vo.launch.py` adds (`config/vo/params/<type>.yaml`): `mipi_csi` (econ IMX568) or `usb_uvc` (ELP AR0234). `x650_config.yaml` is this file with `usb_uvc` |
| `sensors.radar.enabled` / `.type` / `.topic` / `.cfg` | `true` / `ti_mmwave_6843isk` / `/ti_mmwave/radar_scan_pcl` / unset | `.cfg`: the radar profile, a file in `config/radar/` or the driver's `cfg/`; unset = `6843ISK_Standard.cfg`, which both drones fly (section 8) |
| `odometry.visual.enabled` / `.type` / `.sink` | `true` / `visual_odometry` / `mavlink_vision_position_delta` | |
| `flight_controller.model` / `.serial` / `.baud` / `.mavlink_version` | `cube_orange_plus` / `/dev/ttyACM0` / `921600` / `2.0` | `/dev/ttyS1` when wired over UART |
| `flight_controller.bridge.*` | `pose_topic: /vo_debug/odometry` | hw only: parameters of the `fc_bridge` stack's `flight_controller_bridge` over `config/fc_bridge/flight_controller_bridge.hw.yaml`; `pose_topic` is sent to the FC as `VISION_POSITION_ESTIMATE` (`""` sends nothing) |
| `rosbag.record` / `.recording_path` / `.bagname_prefix` / `.compression` | `false` / `TODO` / `rosbag` / `true` | |
| `rosbag.topics` | radar, both detection topics, `vo_debug/odometry`, `/tf`, `/tf_static` | exact-match allowlist |

## 5. Camera: `config/camera/<sensors.camera.type>.yaml` (+ `camera.detected.yaml`)

One file per camera: `mipi_csi.yaml` (the Mistral drone's econ IMX568, below) and `usb_uvc.yaml`
(the x650's ELP AR0234, 1280x960). Each holds the `camera_publisher` parameters and the camera's
`mount`. On hardware the launches take the frame size from it for `neat_inference`,
`radar_camera_fusion` (`source_width` / `source_height`) and `object_tracker` (`image_width` /
`image_height`).

Node: sima-core `camera_publisher`, the single owner of the camera. Feeds both perception and
VO; publishes NV12 and `/camera_info` from the intrinsics below.

| parameter | value | notes |
|---|---|---|
| `camera_name` | resolved on the board | **not in git.** `deploy.sh` runs `cam -l` and writes `camera.detected.yaml`; without it the node's default matches no board we have |
| `frame_id` | `camera_frame` | must equal `neat_inference.frame_id` and the TF parent of the radar extrinsic |
| `image_topic` | `/image_raw` | consumed by perception and VO |
| `camera_info_topic` | `/camera_info` | |
| `width` / `height` / `fps` | `1920` / `1080` / `30` | libcamera caps the sensor at 29.97 fps |
| `fx` / `fy` / `cx` / `cy` | `1606.10` / `1602.05` / `878.79` / `487.19` | **per-unit.** Measured on this drone's camera at 1920x1080 (2026-09-10) |
| `distortion` | `[-0.3535, 0.1516, 0.00056, 0.0018, 0.0]` | k1 k2 p1 p2 k3; real and measured, do not zero |

The same intrinsics are repeated under `visual_odometry.motion.camera` (section 7); keep the two
copies identical.

The camera's mount on the drone is the file's `mount` section, read by the launches, not by a
node: `cam_to_body_roll` / `_pitch` / `_yaw` (rad, optical -> body FRD) and `cam_offset_m`
(m, FRD). The VIO gets it as its extrinsics; target pursuit publishes it on hardware as
`base_link -> camera_frame`, which guidance reads the camera's tilt from. `mipi_csi`: the
Mistral drone, 46° down from flight data, offset from a photo (±5 cm); `usb_uvc`: the x650's
Kalibr camera-IMU calibration, 38.5° down, 3.3° left.

## 6. Perception: `config/perception/radar_fusion*.yaml`

Three nodes from sima-core: `neat_inference` (YOLOv8-seg on the MLA), `radar_camera_fusion`,
`pipeline_health_check`. `radar_fusion.yaml` is the common half; `radar_fusion.hw.yaml` or
`.sim.yaml` is layered last.

### `neat_inference`

| parameter | hw | sim | notes |
|---|---|---|---|
| `model_path` | `resources/yolo_v8m_seg_mpk.tar.gz` | same | relative to the bringup share; provisioned by the manifest |
| `input_mode` | `ros_image` | same | the node has no camera mode |
| `image_topic` | `/image_raw` | `/image_bgr` | |
| `input_encoding` | `nv12` | `bgr8` | fixed at model load, must be declared |
| `source_width` / `source_height` / `fps` | the camera's frame / `30` | `640` / `480` / `10` | must match the publisher; NEAT sizes its input from them. hw: from `config/camera/<type>.yaml` |
| `payload_kind` / `output_mode` | `image` / `ros` | same | |
| `render` / `publish_raw_image` | `false` / `false` | same | `publish_raw_image` must stay false: it would republish onto the node's own input |
| `output_topic` | `simaai/segmentation/detections` | same | |
| `frame_id` | `camera_frame` | same | |
| `model_width` / `model_height` | `640` / `640` | same | model constant |
| `decode_type` | `yolov8_seg` | same | |
| `score_threshold` / `nms_iou_threshold` / `top_k` | `0.5` / `0.5` / `24` | same | node defaults are 0.25 / 0.45 / 100 |

### `radar_camera_fusion`

| parameter | hw | sim | notes |
|---|---|---|---|
| `detections_topic` | `simaai/segmentation/detections` | same | |
| `radar_topic` | `/ti_mmwave/radar_scan_pcl` | same (nothing publishes it) | in sim the node emits vision-only detections with `has_radar: false` |
| `camera_info_topic` | `/camera_info` | same: the simulator publishes it beside `/image_bgr` | |
| `pub_topic` | `/simaai/radar_fusion/detections` | same | |
| `source_width` / `source_height` | the camera's frame | `640` / `480` | the resolution `camera_info`'s K is expressed in. hw: from `config/camera/<type>.yaml` |
| `velocity_field` | `velocity` | same | **VERIFY**: the Doppler field name in this driver build; `""` if none |
| `model_width` / `model_height` | `640` / `640` | same | |
| `mask_threshold` / `use_mask_gate` | `100` / `true` | same | |
| `min_range` / `max_range` | `0.3` / `11.0` m | same | 11.0 is the 6843ISK's physical limit at this sample rate; node default 80 never gated anything |
| `min_points` / `cluster_gate` | `1` / `2.0` | same | |
| `bbox_fallback_margin_px` | `40.0` | same | |
| `queue_size` | `30` | same | |
| `debug_projection` / `debug_projection_topic` | `false` / `/radar_fusion/debug_projection` | same | calibration only; off in flight |
| `residual_log_window` / `residual_log_period_sec` / `residual_assoc_du_max_px` | `200` / `5.0` / `150.0` | same | calibration instrumentation |

Not overridden here, so at node defaults: `accum_buffer_size` (5), `accum_max_age_sec` (0.5).

### `pipeline_health_check`

`stale_threshold_sec: 5.0`, watching the two detection topics, the radar cloud, `image_raw` and
`/camera_info` with their message types.

## 7. Visual odometry: `config/vo/visual_odometry*.yaml`

Node: sima-core `visual_odometry` lifecycle node, running SuperPoint + LightGlue on pyneat.
`visual_odometry.yaml` is the common half; the mode half is layered last.

### Algorithm and output

| parameter | common | hw half | sim half | notes |
|---|---|---|---|---|
| `algorithm` | `keyframe` | | `vio` | sim fuses the simulated IMU and rangefinder |
| `debug.publish_odom` / `debug.odom_topic` | `true` / `vo_debug/odometry` | | | the acceptance gate requires a publisher here |
| `input.image_topic` | | `/image_raw` | `/image_bgr` | |

### Models

| parameter | value | notes |
|---|---|---|
| `superpoint.model_path` | `resources/superpoint_mpk.tar.gz` | |
| `lightglue.model_path` | `resources/lightglue_isaac_qat_mpk.tar.gz` | active since 2026-10-12; `resources/lightglue_mpk.tar.gz` is the commented fallback |
| `lightglue.variant` | `raw` | `posenc` is the rover's alternative, commented out; it needs the rover's pyneat build |
| `lightglue.num_keypoints` | `512` | must match the archive (the retired K=600 archive shares a filename; check md5) |
| `lightglue.match_threshold` | `0.022` | |

### Motion and scale

| parameter | common | hw half | sim half | notes |
|---|---|---|---|---|
| `motion.scale_mode` | `ground_plane` | `parallax` | `parallax` | ground_plane needs the FC's AGL/attitude, so hw falls back until the MAVLink link is verified |
| `motion.min_avg_parallax_px` | `4.0` | | | |
| `motion.max_step_m` | `3.0` | | | |
| `motion.parallax_derotate` / `parallax_direction_normalize` | `true` / `true` | | | |
| `motion.ground.normal_at_cur` | `false` | | | |
| `motion.scale_factor` | `0.3` | | `0.345` | parallax scale only |
| `motion.cam_to_body_roll` / `_pitch` / `_yaw` | `π/2` / `0` / `π/2` | per airframe | `1.309` / `0` / `π/2` | hw: the camera's mount from `config/camera/<sensors.camera.type>.yaml` (with `vio.cam_offset_m`), over `config/vo/params/<type>.yaml` (`vio.cam_time_offset_s`, the IMU noise `vio.gyro_*`, `vio.accel_*`) |
| `motion.camera.fx` / `fy` / `cx` / `cy` | as `config/camera/<type>.yaml` | | `549.749` / `549.749` / `320` / `240` | at the input image's own resolution, not the model's |
| `motion.camera.dist_coeffs` | as `config/camera/<type>.yaml` | | zeros | |
| `keyframe.max_emit_step_m` | | | `1.5` | |

### MAVLink

| parameter | common | hw half | sim half | notes |
|---|---|---|---|---|
| `mavlink.enable` | node default (on) | `false` | | **hw sends nothing to the FC** until the mount, AGL offset and endpoint are measured |
| `mavlink.target_ip` / `target_port` | `127.0.0.1` / `14540` | | `10.42.0.1` / `14540` | **VERIFY**: the MAVLink bridge's UDP endpoint on the board. The sim value is the workstation gateway |
| `mavlink.receive_state` | `true` | | | the bridge must forward ATTITUDE and LOCAL_POSITION_NED back |
| `mavlink.agl_offset_m` | `0.0` | | | **VERIFY**: camera height above the FC's z=0 |

### Sim-only VIO block (`vio.*`)

`imu_topic /sim/imu`, `cam_offset_m [0.12, 0, 0.03]`, `kf_min_interval_s 0.2`,
`kf_max_interval_s 0.5`, `range_topic /sim/rangefinder`, `range_enable true`,
`range_sigma_m 0.03`, `output_rate_hz 30.0`, `batch_enforce_consistency false`.

### Environment set by `launch_vo.launch.py`

`SIMA_ALLOW_INPUTSTREAM_CPU_TO_EV74_COPY=1`, `SIMA_MPK_GRAPH_FUSE_DETESSDEQUANT=0` (required by
the raw LightGlue archives), `SIMA_MPK_EXTRACT_ROOT` (where MPKs are unpacked).

## 8. Radar driver: `config/radar/`

The driver is `ti_mmwave_rospkg`, launched through its own `IWR6843.py` with `rviz:=false` and the
ports `6843ISK_Standard.py` used (`/dev/ttyUSB0`, `/dev/ttyUSB1`).
Its configuration is the sensor `.cfg`, chosen per drone by `sensors.radar.cfg` in the system
config: a file in `config/radar/`, or one in the installed driver's `cfg/`. Two profiles exist:

| | `6843ISK_Standard.cfg` (default, stock, not committed) | `6843ISK_DopC.cfg` (committed) |
|---|---|---|
| `profileCfg` slope / ADC samples | 166 MHz/us / 256 | 166 MHz/us / 64 |
| `frameCfg` loops | 32 | 128 |
| range: max / resolution | 9.0 m / 0.044 m | 9.0 m / 0.176 m |
| Doppler resolution | 0.81 m/s | 0.099 m/s |
| frame rate | 20 FPS | 20 FPS |

The maximum range is set by the 10 MHz IF bandwidth: 10 MHz * c / (2 * slope).

Fusion wants the fine range, so Standard is correct for it. Lines the fusion synchroniser depends on:
`frameCfg` (frame rate, feeds `expect_hz` of the radar topic), `aoaFovCfg` (field of view),
`cfarFovCfg` (range gate: 0.25–9.0 m in Standard and DopC). The drones fly Standard, which is why
fusion's `max_range` is 11.0. Firmware:
`small_obstacle_detection_68xx.bin`, flashed with TI UniFlash; see `provisioning/radar/`.

## 9. Models: `models/models.manifest.yaml`

Provisioned by `deploy.sh` and baked into the release by `gen_release.sh`, which verifies each
md5 and refuses to cut a release on a mismatch.

| manifest entry | archive | installed to (under the install share) | selected by | status |
|---|---|---|---|---|
| `yolov8-seg` | `yolo_v8m_seg_mpk.tar.gz` (38 MB) | `mistral_bringup/config/perception/resources/` | `neat_inference.model_path` | active |
| `superpoint` | `superpoint_mpk.tar.gz` (3 MB) | `visual_odometry/resources/` | `superpoint.model_path` | active |
| `lightglue_isaac_qat` | `lightglue_isaac_qat_mpk.tar.gz` (143 MB), K=512 | `visual_odometry/resources/` | `lightglue.model_path` | **active default** |
| `lightglue_raw` | `lightglue_mpk.tar.gz` (150 MB), K=512, md5 `8de3cb61…` | `visual_odometry/resources/` | commented alternative | shipped as fallback |
| `lightglue_posenc` | `quantized_lightglue_superpoint_600_mpk.tar.gz`, K=600 | `visual_odometry/resources/` | commented `posenc` block | `TODO:` source, skipped |

To swap the LightGlue model: change `lightglue.model_path` (and `variant`, `num_keypoints`,
`match_threshold` together, they are one block) in `config/vo/visual_odometry.yaml`. Both raw
archives share the K=512 contract, so between them only `model_path` changes. Model constants such
as input resolution and dequantization are internal to the node and not externalized.

## 10. Network and DDS

| setting | where | value |
|---|---|---|
| `ROS_DOMAIN_ID` | `deploy.yaml` → `deploy.env` → run wrapper | `25` |
| Fast DDS profile | `config/dds/fastdds_large.xml`, exported by the wrapper | large-message shared-memory profile |
| discovery range | `deploy.sh --isolated` | `LOCALHOST` when set; default is subnet-wide |
| static peers | `deploy.sh --sim-host` | `ROS_STATIC_PEERS=<sim host>` in sim |
| Foxglove bridge | run wrapper `--foxglove` | websocket on `:8765` |

## 11. Acceptance profiles (`acceptance.yaml` in the release)

The rate gate (`check_rates.py`) is pointed at one profile by `deploy.sh`, matching the flags:

| profile | chosen by | what differs |
|---|---|---|
| `hw` | default | all five topics required at their measured rates: image and camera_info 29.98 Hz, segmentation 29.975 Hz, fusion 19.8 Hz, radar 20 Hz |
| `hw_no_radar` | `--no-radar` | radar cloud and fusion output expected `ABSENT` |
| `replay` | `--no-camera` | frames from a bag; `vo_debug/odometry` becomes required |
| `sim` | `--mode sim` | simulator topics; radar and fusion `ABSENT`; VO publisher required |

Defaults: `min_frac 0.7`, `max_gap_frac 10.0`; `hw` uses `warmup_s 8.0`, `window_s 15.0`.

## 12. Values not yet verified on this drone

Collected from the `VERIFY` and `TODO` markers above. Wrong values here do not fail loudly: VO
drifts or its output never reaches the flight controller.

- `radar_camera_fusion.velocity_field`: the Doppler field name in this driver build.
- `cam_offset_m` of the Mistral drone (`config/camera/mipi_csi.yaml`): from a photo, ±5 cm.
- `visual_odometry.mavlink.target_ip` / `target_port`: the MAVLink bridge's endpoint on the board.
- `visual_odometry.mavlink.agl_offset_m`: camera height above the FC's ground reference.
- `mavlink.enable` is forced off on hardware until the three above are measured.
- `sensors.camera.type` and `rosbag.recording_path` in the system config are `TODO`.
- `lightglue_posenc` has no source archive; `deploy.sh` skips it.
- The radar `.cfg` on a board is whatever was last written there; nothing in the release restores it.
