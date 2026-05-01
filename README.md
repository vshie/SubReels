# SubReels Extension

Onboard video recording extension for use with hovering AUV running BlueOS / ArduSub, featuring automated dive mission control with external switch triggering.

## Components

### 1. SubReels Extension
A Python-based web service that provides HTTP endpoints for starting and stopping video recording from USB and IP camera video devices.

**Features:**
- HTTP API for remote recording control
- H.264 video recording with configurable quality settings
- Integration with ArduPilot Lua scripts via HTTP requests

### 2. HAUV Lua Script
An ArduPilot Lua script that controls automated dive missions for a hovering AUV with external switch-based triggering.

**Features:**
- External switch-based mission initiation (GPIO pin 27 Navigator Leak sensor input)
- Automated dive sequence: countdown → descent → hover → ascent → surface
- Configurable parameters for depth, timing, and throttle settings
- Water sampling integration with relay control
- Light control synchronized with dive phases
- Safety abort functionality via switch opening
- Video recording start/stop synchronized with dive phases

## Installation

### Prerequisites
- BlueOS running on compatible hardware ([Installation Guide](https://blueos.cloud/docs/latest/usage/installation/))
- ArduSub firmware with Lua scripting enabled
- USB video device providing H.264 stream on `/dev/video2`

### SubReels Extension Setup

#### Option A — Manual install in BlueOS (recommended)

A new image is published to Docker Hub on every push to `main` via the
`Deploy BlueOS Extension Image` GitHub Action (see `.github/workflows/deploy.yml`).
To install it manually:

1. Open BlueOS in your browser.
2. Go to **Extensions → Installed Extensions** and click the **+** button
   in the bottom-right.
3. Fill in the dialog:
   - **Extension Identifier:** `vshie.subreels`
   - **Extension Name:** `SubReels`
   - **Docker image:** `vshie/subreels`
   - **Docker tag:** `main`
   - **Permissions:** copy and paste the JSON block below verbatim.

   ```json
   {
     "ExposedPorts": {
       "5423/tcp": {}
     },
     "HostConfig": {
       "Binds": [
         "/usr/blueos/extensions/subreels:/app/videorecordings",
         "/dev/video2:/dev/video2"
       ],
       "ExtraHosts": ["host.docker.internal:host-gateway"],
       "PortBindings": {
         "5423/tcp": [
           {
             "HostPort": ""
           }
         ]
       },
       "NetworkMode": "host",
       "Privileged": true
     }
   }
   ```

4. Click **Create**. BlueOS will pull the image from Docker Hub and start the
   container.
5. Once it shows as running, click the SubReels icon in the BlueOS sidebar
   to open the dashboard.

The permissions block above is identical to the `LABEL permissions` baked
into the Docker image and tells BlueOS to:

- **Bind `/usr/blueos/extensions/subreels` → `/app/videorecordings`** so
  recordings persist on the BlueOS host and are visible in the BlueOS file
  browser at `http://<host>:7777/files/extensions/subreels/`.
- **Bind `/dev/video2` → `/dev/video2`** so the container can open the USB
  camera. Requires that `/dev/video2` is *not* configured on the BlueOS
  Video Streams page (see *Camera setup gotchas* below).
- **Use host networking, privileged mode, and `host.docker.internal`** so
  the container can reach MAVLink2REST on the BlueOS host for telemetry.

#### Option B — Build a local image

If you want to test changes before they hit Docker Hub, build the image
locally and point BlueOS at it:

```bash
git clone https://github.com/vshie/SubReels.git
cd SubReels
docker build -t subreels:local .
```

Then in the **Installed Extensions → +** dialog, set **Docker image** to
`subreels` and **Docker tag** to `local` (and reuse the same permissions
JSON from Option A).

#### Camera setup gotchas

- **`/dev/video2` must NOT be added to the BlueOS Video Streams page.**
  SubReels needs exclusive access to the USB camera while recording.
  As a result, you will not see a Cockpit preview from the USB camera
  while this extension is installed — that is intentional, since for a
  tetherless dive there is no surface to preview to anyway. The RTSP H.265
  RadCam stream is unaffected and can be previewed normally.
- The RTSP endpoint is currently hardcoded in `app/main.py`
  (`RTSP_H265_ENDPOINT`, default `rtsp://admin:blue@192.168.2.10:554/stream_0`).
  If your IP camera lives at a different address or uses different
  credentials, edit that constant and rebuild the image.

### HAUV Lua Script Setup

1. **Enable ArduPilot Lua scripting:**
   - Set parameter `SCR_ENABLE` to `1` (true)
   - Restart the autopilot for changes to take effect

2. **Upload the script:**
   - Copy `HAUV.lua` to the ArduPilot scripts directory
   - The script will automatically execute on autopilot startup

3. **Hardware connections:**
   - Connect external switch to GPIO pin 27 (PWM0/RGB port on Navigator)
   - Switch should connect to 3.3V and signal pin
   - Close switch to initiate dive mission
   - Open switch during mission to abort

## Configuration

### HAUV Script Parameters

The script includes configurable parameters accessible through Mission Planner or similar GCS:

- `HOVER_DELAY_S`: Countdown delay before dive (default: 30 seconds)
- `HOVER_LIGHT_D`: Depth to turn on lights (default: 7.0m)
- `HOVER_HOVER_M`: Hover duration in minutes (default: 1.0)
- `HOVER_SURF_D`: Surface threshold depth (default: 2.0m)
- `HOVER_MAX_AH`: Maximum amp-hours before abort (default: 12.0)
- `HOVER_MIN_V`: Minimum battery voltage (default: 13.0V)
- `HOVER_REC_DEPTH`: Depth to start video recording (default: 5.0m)
- `HOVER_T_DEPTH`: Target maximum depth (default: 60m)
- `HOVER_D_THRTL`: Descent throttle setting (default: 1750)
- `HOVER_A_THRTL`: Ascent throttle setting (default: 1460)
- `HOVER_SIM_MODE`: Simulation mode (0=normal, 1=simulation)

### SubReels Configuration

The extension uses HTTP endpoints:
- `GET /start` - Start video recording
- `GET /stop` - Stop video recording
- Default port: 5423

## Mission Flow

1. **Standby**: Vehicle waits with switch open
2. **Countdown**: Switch closed triggers countdown period
3. **Descent**: Vehicle descends to target depth with video recording
4. **Hover**: Vehicle hovers at target depth for specified duration
5. **Ascent**: Vehicle ascends to surface with lights controlled by depth
6. **Surface**: Vehicle completes mission and disarms

## Safety Features

- **External switch abort**: Opening switch during mission triggers immediate abort
- **Battery monitoring**: Mission aborts if voltage or amp-hours exceed limits
- **Timeout protection**: Missions have configurable time limits
- **Depth collision detection**: Automatic hover depth adjustment on impact detection
- **Automatic disarm**: Vehicle disarms on mission completion or abort

## Troubleshooting

### Common Issues

1. **Script not executing:**
   - Verify `SCR_ENABLE` parameter is set to `1`
   - Restart autopilot after parameter change
   - Check script file is in correct directory

2. **Video recording fails:**
   - Ensure `/dev/video2` streams are disabled in BlueOS Video Streams
   - Verify video device provides H.264 stream
   - Check extension web service is running on port 5423

3. **Switch not responding:**
   - Verify GPIO pin 27 connection
   - Check switch wiring to 3.3V and signal pin
   - Monitor GCS messages for switch state updates

## Hardware

CAD files for the Profiling AUVE is available:
- [Onshape CAD Document](https://cad.onshape.com/documents/e4693243722d954d549cf47c/w/2125e0004d02499999f2c26f/e/567b98a997673cb9745957bb?renderMode=0&uiState=68d9aa5e71f48e4fab9e347f)

## Development

This project is in active development. For issues, feature requests, or contributions, please refer to the project repository.

## License

See LICENSE file for details.
