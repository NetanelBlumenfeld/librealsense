The librealsense repository ships a set of small command-line tools, next to the graphical [[Realsense Viewer Manual]] and [[Depth Quality Tool]]. They list connected cameras, record and convert recordings, collect frame statistics, read firmware logs, send debug commands to the camera, and measure processing-block performance. This page explains how the tools are built and documents every flag that each tool's command-line parser accepts.

Tools covered elsewhere:

- `rs-fw-update`: see [[Firmware Update]].
- `realsense-viewer`: see [[Realsense Viewer Manual]].
- `rs-depth-quality`: see [[Depth Quality Tool]].
- The DDS tools in [`tools/dds`](https://github.com/realsenseai/librealsense/tree/development/tools/dds): see [[Ethernet and DDS Cameras]].
- The RUM development server (`rum_dev_server.py`) is documented below. For the usage-statistics feature itself, see [[Usage Statistics RUM]].

The source folder for all tools is [`tools/`](https://github.com/realsenseai/librealsense/tree/development/tools).

## Building the tools

CMake adds the `tools` folder when `BUILD_EXAMPLES` or `BUILD_TOOLS` is on (root [`CMakeLists.txt`](https://github.com/realsenseai/librealsense/blob/development/CMakeLists.txt)). Both options default to `ON` in [`CMake/lrs_options.cmake`](https://github.com/realsenseai/librealsense/blob/development/CMake/lrs_options.cmake). [`tools/CMakeLists.txt`](https://github.com/realsenseai/librealsense/blob/development/tools/CMakeLists.txt) then decides which tools are built:

| Tool (executable) | Built when |
|---|---|
| `rs-convert`, `rs-enumerate-devices`, `rs-fw-logger`, `rs-terminal`, `rs-record`, `rs-fw-update`, `rs-embed` | `BUILD_TOOLS` is on |
| DDS tools | `BUILD_TOOLS` and `BUILD_WITH_DDS` are on |
| `rs-data-collect`, `realsense-viewer`, `rs-depth-quality`, `rs-rosbag-inspector`, `rs-benchmark` | `BUILD_EXAMPLES` and `BUILD_GRAPHICAL_EXAMPLES` are on |

`BUILD_WITH_DDS` defaults to `OFF`. `BUILD_GRAPHICAL_EXAMPLES` defaults to `ON` and also turns on `BUILD_GLSL_EXTENSIONS`. On Android, [`CMake/android_config.cmake`](https://github.com/realsenseai/librealsense/blob/development/CMake/android_config.cmake) turns off `BUILD_TOOLS`, `BUILD_EXAMPLES` and `BUILD_GRAPHICAL_EXAMPLES`, so no tools are built there.

For example, to build only the console tools without examples or graphical applications:

```bash
cmake .. -DBUILD_EXAMPLES=false -DBUILD_GRAPHICAL_EXAMPLES=false -DBUILD_TOOLS=true
```

`make install` copies the tools into `${CMAKE_INSTALL_BINDIR}`. `rs-embed` is the exception: it goes to `${CMAKE_INSTALL_PREFIX}/bin`. For all build options, see [[Build Configuration]].

The RUM development server is a Python script. It is not part of the CMake build; you run it directly with Python 3.

## Common flags

Most tools build their parser with `rs2::cli` from [`common/cli.h`](https://github.com/realsenseai/librealsense/blob/development/common/cli.h), which is based on TCLAP. Every such parser registers these flags, in this order, before the tool's own flags:

1. **TCLAP built-ins**: `-h`/`--help` (prints usage and exits), `--version` (prints the version and exits) and `--`/`--ignore_rest` (ignores all labeled arguments after it). For `rs2::cli` tools, `--version` prints the librealsense version (`RS2_API_FULL_VERSION_STR`).
2. **`--debug`**: logs librealsense messages to the console at debug level. The flag exists only when the library is built with `BUILD_EASYLOGGINGPP` (default `ON`). Without `--debug`, a tool logs only errors to the console. `rs-convert` and `rs-data-collect` log warnings and errors.
3. **DDS flags**: `--eth`, `--eth-only`, `--no-eth` and `--domain-id <0-232>`. `rs2::cli` registers these flags. `rs2::cli_no_dds`, which `rs-fw-logger` uses, does not. The flags have an effect only when the library is built with `BUILD_WITH_DDS`, and otherwise are ignored:
   - `--eth` enables DDS device detection, even when the configuration disables it.
   - `--eth-only` uses only DDS (software) devices, with no USB or MIPI devices.
   - `--no-eth` disables DDS device detection.
   - `--domain-id` sets the DDS domain. Values outside `[0, 232]` are rejected.
   - `--eth`, `--eth-only` and `--no-eth` are mutually exclusive. `--domain-id` cannot be combined with `--no-eth`.

`rs-embed` uses a plain `TCLAP::CmdLine` and has only the TCLAP built-ins. `rs-benchmark` has only the shared flags. `rs-rosbag-inspector` does not parse command-line arguments.

## rs-enumerate-devices

`rs-enumerate-devices` prints information about the connected RealSense devices. By default, it prints the camera info of each device (name, serial number, firmware version and other `rs2_camera_info` fields) and the stream profiles that each sensor supports. Profiles with the same stream, resolution and format are grouped on one line with their frame rates separated by `/`.

Source: [`tools/enumerate-devices/rs-enumerate-devices.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/enumerate-devices/rs-enumerate-devices.cpp) · [readme](https://github.com/realsenseai/librealsense/blob/development/tools/enumerate-devices/readme.md)

```bash
rs-enumerate-devices -s          # one table line per device: name, serial number, firmware version
rs-enumerate-devices -o -c       # add all supported options and the intrinsics/extrinsics
rs-enumerate-devices -p rec.db3  # also load a recording as a playback device and list it
```

- `-s` prints only the summary table. `-S` prints the table followed by the camera info of each device. Both modes suppress stream profiles, options and calibration data. If you combine `-s` or `-S` with `-o` or `-c`, the tool prints a warning and ignores `-o` and `-c`.
- `-d` marks default profiles with `+` in the profile list. With `-s` or `-S`, it prints a "Default streams" list for each device instead.
- `-v` prints each profile on its own line and adds its unique ID and stream index.
- `--sw-only` lists only software devices (playback, DDS), not USB/HID devices. If none is found at first, the tool retries 3 times, once per second.
- The tool enables DDS device detection unless you pass `--no-eth`. This has an effect only in builds with `BUILD_WITH_DDS`.

<!-- wikigen:table-rs-enumerate-devices -->
| Flag | Argument | Description | Default |
|---|---|---|---|
| `-h`, `--help` | — | Print usage and exit | — |
| `--version` | — | Print the librealsense version and exit | — |
| `--`, `--ignore_rest` | — | Ignore the rest of the labeled arguments | — |
| `--debug` | — | Turn on librealsense debug logs (only with `BUILD_EASYLOGGINGPP`) | off |
| `--eth` | — | Detect DDS devices even if disabled in the configuration | off |
| `--eth-only` | — | Use only DDS devices | off |
| `--no-eth` | — | Do not detect DDS devices | off |
| `--domain-id` | `0-232` | DDS domain ID | `0` |
| `-s`, `--short` | — | One-line summary per device (name, serial number, firmware version) | off |
| `-S`, `--compact` | — | Summary table plus the camera info of each device | off |
| `-o`, `--option` | — | Show all supported options per sensor (min, max, step, default) | off |
| `-c`, `--calib_data` | — | Show intrinsics and extrinsics of all sensors | off |
| `--sw-only` | — | Show only software devices (playback, DDS, etc.), not USB/HID devices | off |
| `--format` | `raw/basic/FULL` | Format conversion: `full`, `basic` or `raw` (lowercase values) | `full` |
| `-v`, `--verbose` | — | One line per profile, with unique ID and stream index | off |
| `-d`, `--defaults` | — | Mark default stream profiles; with `-s`/`-S`, list the default streams | off |
| `-p`, `--playback_device` | `path` | Load a recording file as a playback device and include it in the output | — |
<!-- /wikigen:table-rs-enumerate-devices -->

## rs-record

`rs-record` starts a pipeline with its default configuration, records the streams to a file and stops after a set time. While it records, it prints `Recording t = <N>s` once per second, and it prints `Finished` when done. The writer depends on `BUILD_ROSBAG2` (default `ON`). With it, recordings use the rosbag2 `.db3` format. Without it, the legacy writer is used, and recording to a `.db3` file fails. For the recording formats and playback, see [[Record and Playback]].

Source: [`tools/recorder/rs-record.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/recorder/rs-record.cpp) · [readme](https://github.com/realsenseai/librealsense/blob/development/tools/recorder/readme.md)

```bash
rs-record -f ./capture.db3 -t 30
```

<!-- wikigen:table-rs-record -->
| Flag | Argument | Description | Default |
|---|---|---|---|
| `-h`, `--help` | — | Print usage and exit | — |
| `--version` | — | Print the librealsense version and exit | — |
| `--`, `--ignore_rest` | — | Ignore the rest of the labeled arguments | — |
| `--debug` | — | Turn on librealsense debug logs (only with `BUILD_EASYLOGGINGPP`) | off |
| `--eth` | — | Detect DDS devices even if disabled in the configuration | off |
| `--eth-only` | — | Use only DDS devices | off |
| `--no-eth` | — | Do not detect DDS devices | off |
| `--domain-id` | `0-232` | DDS domain ID | `0` |
| `-t`, `--Time` | `seconds` | Recording duration in seconds (fractions allowed) | `10` |
| `-f`, `--FullFilePath` | `path` | Output file (required) | — |
<!-- /wikigen:table-rs-record -->

## rs-convert

`rs-convert` reads a recording file and writes its frames in other formats. It can also convert a legacy ROS1 `.bag` file to `.db3`. You can use several output converters in one run. Each converter writes one file per frame, named `<path>_<stream name>_<timestamp>.<ext>`, plus a `_metadata_` text file. PLY output is the exception: it writes `<path>_<timestamp>.ply` from synchronized depth and color frames. After the conversion, the tool prints statistics for each converter.

Source: [`tools/convert/rs-convert.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/convert/rs-convert.cpp) · [converters](https://github.com/realsenseai/librealsense/tree/development/tools/convert/converters) · [readme](https://github.com/realsenseai/librealsense/blob/development/tools/convert/readme.md)

```bash
# Legacy .bag -> .db3 (other output flags are ignored in this mode)
rs-convert -i recording.bag -D recording.db3

# PNG and RAW frames of the depth stream only, frames 100..200
rs-convert -i recording.db3 -p out/png -r out/raw -d -f 100 -t 200
```

- At least one output flag is required (`-p`, `-v`, `-V`, `-r`, `-l`, `-b`, `-T` or `-D`). Otherwise the tool exits with `output not defined`.
- `-d` and `-c` limit the PNG, RAW, CSV and 3D-CSV converters to depth or color frames. If both flags are given, `-d` wins. `-b` always writes depth frames. `-l` always uses depth and color together.
- `-T` prints one line per frame to stdout. It also suppresses the progress percentage and statistics of the non-PLY converters.

<!-- wikigen:table-rs-convert -->
| Flag | Argument | Description | Default |
|---|---|---|---|
| `-h`, `--help` | — | Print usage and exit | — |
| `--version` | — | Print the librealsense version and exit | — |
| `--`, `--ignore_rest` | — | Ignore the rest of the labeled arguments | — |
| `--debug` | — | Turn on librealsense debug logs (only with `BUILD_EASYLOGGINGPP`) | off |
| `--eth` | — | Detect DDS devices even if disabled in the configuration | off |
| `--eth-only` | — | Use only DDS devices | off |
| `--no-eth` | — | Do not detect DDS devices | off |
| `--domain-id` | `0-232` | DDS domain ID | `0` |
| `-i`, `--input` | `ros-bag-file` | Input recording file (required) | — |
| `-t`, `--last-framenumber` | `frame` | Skip frames whose frame number is greater than this value | — |
| `-f`, `--first-framenumber` | `frame` | Skip frames whose frame number is less than this value | — |
| `-e`, `--end-time` | `seconds` | Skip frames after this time (the first frame is at time 0) | — |
| `-s`, `--start-time` | `seconds` | Skip frames before this time (the first frame is at time 0) | — |
| `-p`, `--output-png` | `png-path` | Write PNG files with this path prefix | — |
| `-v`, `--output-csv` | `csv-path` | Write CSV files with this path prefix | — |
| `-V`, `--output-3d-csv` | `3d-csv-path` | Write 3D CSV files with this path prefix | — |
| `-r`, `--output-raw` | `raw-path` | Write RAW files with this path prefix | — |
| `-l`, `--output-ply` | `ply-path` | Write PLY point clouds (synchronized depth and color) with this path prefix | — |
| `-b`, `--output-bin` | `bin-path` | Write BIN depth-matrix files with this path prefix | — |
| `-d`, `--depth` | — | Convert depth frames only | off (all streams) |
| `-c`, `--color` | — | Convert color frames only | off (all streams) |
| `-T`, `--output-text` | — | Print a text line per frame to stdout | off |
| `-D`, `--output-db3` | `db3-path` | Convert the legacy `.bag` input to a `.db3` file | — |
<!-- /wikigen:table-rs-convert -->

## rs-data-collect

`rs-data-collect` streams a set of profiles from one device through the low-level sensor API and writes per-frame statistics to a CSV-like file. For each frame, it records the stream type, index, frame number, hardware timestamp and host timestamp, and the motion or pose values for IMU and pose streams. It does not save image content. The tool uses the first device found, or the one given with `-s`, and waits for a device if none is connected.

Source: [`tools/data-collect/rs-data-collect.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/data-collect/rs-data-collect.cpp) · [readme](https://github.com/realsenseai/librealsense/blob/development/tools/data-collect/readme.md)

```bash
rs-data-collect -c ./data_collect.cfg -f ./log.csv -t 60 -m 1000
```

The configuration file (`-c`) is required. Each line is `STREAM,WIDTH,HEIGHT,FPS,FORMAT[,STREAM_INDEX]`, and lines that do not start with a letter are skipped:

```
DEPTH,640,480,30,Z16,0
INFRARED,640,480,30,Y8,1
COLOR,640,480,30,RGB8,0
ACCEL,1,1,63,MOTION_XYZ32F
GYRO,1,1,200,MOTION_XYZ32F
```

The stop conditions depend on which flags you set:

- Neither `-t` nor `-m`: the tool stops after 100 frames per stream.
- Only `-t`: the tool stops after the timeout.
- Only `-m`: the tool stops after that many frames per stream.
- Both: the tool stops at whichever comes first.

<!-- wikigen:table-rs-data-collect -->
| Flag | Argument | Description | Default |
|---|---|---|---|
| `-h`, `--help` | — | Print usage and exit | — |
| `--version` | — | Print the librealsense version and exit | — |
| `--`, `--ignore_rest` | — | Ignore the rest of the labeled arguments | — |
| `--debug` | — | Turn on librealsense debug logs (only with `BUILD_EASYLOGGINGPP`) | off |
| `--eth` | — | Detect DDS devices even if disabled in the configuration | off |
| `--eth-only` | — | Use only DDS devices | off |
| `--no-eth` | — | Do not detect DDS devices | off |
| `--domain-id` | `0-232` | DDS domain ID | `0` |
| `-t`, `--Timeout` | `seconds` | Stop after this many seconds | not set (see stop conditions) |
| `-m`, `--MaxFrames_Number` | number | Stop after this many frames per stream | not set (100 frames when `-t` is also unset) |
| `-f`, `--FullFilePath` | `path` | Output file | `frames_data.csv` |
| `-c`, `--ConfigurationFile` | `path` | Stream configuration file (required) | — |
| `-s`, `--SerialNumber` | `serial_number` | Serial number of the device to use | first device |
<!-- /wikigen:table-rs-data-collect -->

## rs-fw-logger

`rs-fw-logger` collects the firmware log of a camera. Use it to capture firmware-side information when you report an issue. The tool waits until a device (or the device with the given serial number) is connected, then polls its firmware log until the device is disconnected. When the device reconnects, logging continues. If the device does not support firmware logging, the tool exits with an error.

Without `-l`, each log entry is printed as raw hex bytes after `FW_Log_Data:`. With a firmware-events XML file, each entry is parsed into timestamp, sequence ID, severity, thread, module, file, line and message. With `-f`, the tool reads the flash log instead and exits once all entries have been read. The tool also writes librealsense warnings to `librealsense.log` in the current folder.

Source: [`tools/fw-logger/rs-fw-logger.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/fw-logger/rs-fw-logger.cpp) · [readme](https://github.com/realsenseai/librealsense/blob/development/tools/fw-logger/readme.md)

```bash
rs-fw-logger -l HWLoggerEvents.xml -o fw_log.txt
rs-fw-logger -s <serial-number> -f
```

`rs-fw-logger` uses `rs2::cli_no_dds`, so it has no DDS flags.

<!-- wikigen:table-rs-fw-logger -->
| Flag | Argument | Description | Default |
|---|---|---|---|
| `-h`, `--help` | — | Print usage and exit | — |
| `--version` | — | Print the librealsense version and exit | — |
| `--`, `--ignore_rest` | — | Ignore the rest of the labeled arguments | — |
| `--debug` | — | Turn on librealsense debug logs (only with `BUILD_EASYLOGGINGPP`) | off |
| `-s`, `--sn` | `serial-number` | Serial number of the camera to log | first device |
| `-l`, `--load` | `path` | Firmware-events XML file used to parse the log | — (raw hex output) |
| `-o`, `--out` | `path` | Output file | — (stdout) |
| `-p`, `--polling_interval` | `milliseconds` | Polling interval; values outside 25–300 fall back to the default | `100` |
| `-f`, `--flash` | — | Read the flash log instead of the live firmware log | off |
<!-- /wikigen:table-rs-fw-logger -->

## rs-terminal

`rs-terminal` sends debug-protocol commands to the camera firmware. It has two modes:

- **Hex mode** (no `-l`): you type raw command bytes in hexadecimal, separated by spaces, and the tool prints the raw response.
- **XML mode** (`-l <commands.xml>`): you type commands by name with their parameters, using a command-definition XML file. The tool offers auto-completion and decodes the response.

In the interactive prompt (`#>`), `exit` quits the tool and `next` waits for another device to connect. The tool also writes librealsense warnings to `librealsense.log`.

Source: [`tools/terminal/rs-terminal.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/terminal/rs-terminal.cpp) · [readme](https://github.com/realsenseai/librealsense/blob/development/tools/terminal/readme.md)

```bash
rs-terminal                                  # interactive hex mode, single device
rs-terminal -l cmd.xml -c sequence.txt       # run named commands from a script file
rs-terminal -n <serial-number> -s "14 00 ab cd 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00"
```

Device selection uses this order of precedence: `-a`, then `-n`, then `-d`. If you give none of them and more than one camera is connected, the tool asks you to choose one and exits. Command sources also have an order of precedence: `-s`, then `-r`, then `-c`, then the interactive prompt. `-c` needs `-l`, because script lines are parsed as named commands.

<!-- wikigen:table-rs-terminal -->
| Flag | Argument | Description | Default |
|---|---|---|---|
| `-h`, `--help` | — | Print usage and exit | — |
| `--version` | — | Print the librealsense version and exit | — |
| `--`, `--ignore_rest` | — | Ignore the rest of the labeled arguments | — |
| `--debug` | — | Turn on librealsense debug logs (only with `BUILD_EASYLOGGINGPP`) | off |
| `--eth` | — | Detect DDS devices even if disabled in the configuration | off |
| `--eth-only` | — | Use only DDS devices | off |
| `--no-eth` | — | Do not detect DDS devices | off |
| `--domain-id` | `0-232` | DDS domain ID | `0` |
| `-l`, `--load` | `path` | Command-definition XML file (enables XML mode) | — (hex mode) |
| `-d`, `--deviceId` | number | Index of the device to use | `0` (used only when set) |
| `-n`, `--serialNum` | `serial-number` | Serial number of the device to use | — |
| `-a`, `--allDevices` | — | Send the commands to all connected devices | off |
| `-s`, `--send` | `hex-data` | Send one raw hex command and exit | — |
| `-r`, `--raw` | `path` | Run a script file of raw hex commands, one per line, and exit | — |
| `-c`, `--cmd` | `path` | Run a script file of named commands, one per line, and exit | — |
| `-v`, `--verbose` | — | Print the raw command and response buffers of named (XML-mode) commands | off |
<!-- /wikigen:table-rs-terminal -->

## rs-benchmark

`rs-benchmark` measures how long librealsense processing blocks take on the current machine. It needs a device with a depth stream and either a color or an infrared stream. It captures about 5 seconds of frames from each stream, runs each processing block on them, and prints Markdown tables with the median, mean, standard deviation and maximum times in milliseconds. The tables also give an estimated maximum frame rate for each block.

The CPU blocks tested are `colorizer`, `pointcloud`, `spatial_filter`, `temporal_filter`, `disparity_transform`, `threshold_filter` and `decimation_filter` on depth, and `yuy_decoder` on YUYV streams. On platforms other than macOS, the tool also tests the GLSL versions `gl::colorizer` and `gl::pointcloud` on depth and `gl::yuy_decoder` on YUYV streams.

Source: [`tools/benchmark/rs-benchmark.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/benchmark/rs-benchmark.cpp) · [readme](https://github.com/realsenseai/librealsense/blob/development/tools/benchmark/readme.md)

```bash
rs-benchmark > benchmark.md
```

<!-- wikigen:table-rs-benchmark -->
| Flag | Argument | Description | Default |
|---|---|---|---|
| `-h`, `--help` | — | Print usage and exit | — |
| `--version` | — | Print the librealsense version and exit | — |
| `--`, `--ignore_rest` | — | Ignore the rest of the labeled arguments | — |
| `--debug` | — | Turn on librealsense debug logs (only with `BUILD_EASYLOGGINGPP`) | off |
| `--eth` | — | Detect DDS devices even if disabled in the configuration | off |
| `--eth-only` | — | Use only DDS devices | off |
| `--no-eth` | — | Do not detect DDS devices | off |
| `--domain-id` | `0-232` | DDS domain ID | `0` |
<!-- /wikigen:table-rs-benchmark -->

## rs-embed

`rs-embed` is a build helper that turns a resource file into a C++ header, for example to embed 3D device models in the Viewer:

- **`.obj` input**: the tool reads vertices and faces, compresses them with LZ4 and generates a `<name>_obj_data` array plus an `uncompress_<name>_obj(...)` function.
- **`.png` input**: the tool writes the file bytes as a `<name>_png_data` array plus `<name>_png_size`.

The tool ignores other file extensions.

Source: [`tools/embed/rs-embed.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/embed/rs-embed.cpp) · [readme](https://github.com/realsenseai/librealsense/blob/development/tools/embed/readme.md)

```bash
rs-embed -i inputs/d435.obj -o d435.h -n d435
```

`rs-embed` uses a plain `TCLAP::CmdLine`, so it has no `--debug` or DDS flags. Its `--version` prints the tool's own version string (`0.0.0.2`).

<!-- wikigen:table-rs-embed -->
| Flag | Argument | Description | Default |
|---|---|---|---|
| `-h`, `--help` | — | Print usage and exit | — |
| `--version` | — | Print the rs-embed version and exit | — |
| `--`, `--ignore_rest` | — | Ignore the rest of the labeled arguments | — |
| `-i`, `--input` | `input-file` | Input `.obj` or `.png` file (required) | — |
| `-o`, `--output` | `output-file` | Output header file | — |
| `-n`, `--name` | `object-name` | Name used for the generated array and function | — |
<!-- /wikigen:table-rs-embed -->

## rs-rosbag-inspector

`rs-rosbag-inspector` is a GUI application for inspecting legacy ROS1 `.bag` recordings. You load files by dragging them onto the window or with **File > Load File...**. The tool shows the topics and messages of each file. It takes no command-line arguments. To inspect `.db3` recordings, the [readme](https://github.com/realsenseai/librealsense/blob/development/tools/rosbag-inspector/readme.md) suggests a third-party application that supports `.db3`. To convert a legacy `.bag` file, use `rs-convert -i <file>.bag -D <file>.db3` (see [rs-convert](#rs-convert)).

Source: [`tools/rosbag-inspector/rs-rosbag-inspector.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/rosbag-inspector/rs-rosbag-inspector.cpp) · [legacy readme](https://github.com/realsenseai/librealsense/blob/development/tools/rosbag-inspector/readme-legacy-ros1.md)

## RUM development server (rum_dev_server.py)

`rum_dev_server.py` is a minimal local server for developers who work on the usage-statistics uploader. It is not the production server. It listens on `127.0.0.1` and accepts `POST /v1/rum`. Other paths get a `404` response. The server saves each request body under the output folder as `rum-<date>-<time>-<NNN>.json` (pretty-printed if the body is JSON), prints a short summary of the report and replies `200` with `{"status":"ok"}`.

The uploader sends to the production endpoint (`RUM_ENDPOINT` in [`common/rum-uploader/rum-uploader.cpp`](https://github.com/realsenseai/librealsense/blob/development/common/rum-uploader/rum-uploader.cpp)). To send reports to this server instead, change `RUM_ENDPOINT` to `http://127.0.0.1:8080/v1/rum` and rebuild. For what is collected and how consent works, see [[Usage Statistics RUM]].

Source: [`tools/rum-uploader/dev-server/rum_dev_server.py`](https://github.com/realsenseai/librealsense/blob/development/tools/rum-uploader/dev-server/rum_dev_server.py)

```bash
python3 tools/rum-uploader/dev-server/rum_dev_server.py --port 8080 --dir received
```

The script uses Python `argparse`, which adds `-h`/`--help`. It has no `--version` flag.

<!-- wikigen:table-rum_dev_server -->
| Flag | Argument | Description | Default |
|---|---|---|---|
| `-h`, `--help` | — | Print usage and exit | — |
| `--port` | integer | TCP port to listen on (on `127.0.0.1`) | `8080` |
| `--dir` | folder | Folder where received reports are saved | `received` |
<!-- /wikigen:table-rum_dev_server -->

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
