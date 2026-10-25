The Depth Quality Tool (`rs-depth-quality`) is a graphical application from librealsense that measures the depth quality of a stereo depth camera pointed at a flat target such as a wall. It fits a plane to the depth data inside a region of interest (ROI) and reports fill rate, spatial noise, sub-pixel accuracy, temporal noise and, when you enter a measured ground-truth distance, Z accuracy. Use it to compare camera settings and presets, check a camera after calibration, or record metrics and raw frames for offline analysis.

![Depth Quality Tool](res/depth-quality-glimpse.gif)

## Building and running the tool

The tool's source is in [`tools/depth-quality`](https://github.com/realsenseai/librealsense/tree/development/tools/depth-quality). CMake builds the `rs-depth-quality` executable when both of these options are `ON` (both default to `ON` in [`CMake/lrs_options.cmake`](https://github.com/realsenseai/librealsense/blob/development/CMake/lrs_options.cmake)):

| CMake option | Default | Role |
|---|---|---|
| `BUILD_EXAMPLES` | `ON` | [`tools/CMakeLists.txt`](https://github.com/realsenseai/librealsense/blob/development/tools/CMakeLists.txt) adds the graphical tools only inside `if(BUILD_EXAMPLES)` |
| `BUILD_GRAPHICAL_EXAMPLES` | `ON` | Builds the Viewer and the Depth Quality Tool; turns on `BUILD_GLSL_EXTENSIONS` if it is off |

`BUILD_TOOLS` on its own does **not** build the Depth Quality Tool. If `BUILD_EXAMPLES` is `OFF`, only the command-line tools are built. See [[Build Configuration]] for all CMake options.

The install step copies `rs-depth-quality` to the binary directory (`CMAKE_INSTALL_BINDIR`). On Windows it is built as a GUI application (no console window).

To start it, connect a camera and run:

```sh
rs-depth-quality
```

The window title is **Depth Quality Tool**. When it starts, the tool:

1. Opens the depth and infrared streams with their default profiles through a pipeline. If no device is connected, the tool keeps showing its splash screen and retries until one is available. If the device cannot be started, it shows `Device is not functional or busy!`.
2. If the device supports advanced mode (see [[D400 Advanced Mode]]) and advanced mode is off, it shows `Toggling device into Advanced Mode...` and switches the device into advanced mode. DDS devices are not switched.
3. Shows the depth stream in the 3D view, with the depth colorizer set to a fixed 0.3 m – 2.7 m range and histogram equalization off.

If the active camera is disconnected, the tool stops streaming and resets. It starts again when a device is available.

## Setting up the target

The tool needs a flat surface that fills the ROI. Point the camera at a flat, non-reflective wall and hold it steady.

While it cannot fit a plane to the ROI, the tool shows `Please point the camera to a flat Wall / Surface!` over the view and hides every metric that needs a plane fit. Once a plane is found, the tool draws it in the 3D view and guides you with these messages (all from [`depth-quality-model.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/depth-quality-model.cpp)):

| Condition | Message | Second line |
|---|---|---|
| Angle to the plane above 10 degrees, camera turned one way | `Rotate the camera slightly Right` / `Left` / `Up` / `Down` | `Recommended angle: < 3 degrees` |
| Distance below 300 mm | `Move the camera further Away` | `Recommended distance: 0.3m-2m from the target` |
| Distance above 2000 mm | `Move the camera Closer to the wall` | `Recommended distance: 0.3m-2m from the target` |

The **Guides** panel on the right of the view shows:

- **Orientation**: the angle between the camera and the plane in degrees, with a gimbal-style indicator.
- **Distance**: the distance to the plane in mm, on a scale where the 300–2000 mm band is shown in green.

Tips:

- Aim for the recommended angle of less than 3 degrees. The **Angle** field in the control panel shows the current value.
- Keep the whole ROI on the flat target. Objects or edges inside the ROI change the fitted plane and the metrics.

## Choosing the region of interest and stream settings

The **Configuration** section of the left panel has these controls:

| Control | What it does |
|---|---|
| Resolution / FPS | Selects the depth stream profile. The tool applies the same resolution to the infrared stream when the device can stream it; otherwise the infrared stream keeps its previous resolution. If the combination is not supported the tool shows `Selected configuration is not supported!`. |
| **Region of Interest** | Size of the ROI as a percentage of the frame width and height: `80%`, `60%`, `40%` (default) or `20%`. The ROI is always centred in the frame. |
| **IR Reflectivity** | Shown only when the depth sensor supports the IR reflectivity option. It can be turned on only with a 20% ROI; otherwise the tool shows `Please set 'VGA' resolution, 'Max Range' preset and 20% ROI before enabling IR Reflectivity`. While it is on, the ROI cannot be changed (`ROI cannot be changed while IR Reflectivity is enabled`). |
| **Distance** | Distance from the camera to the fitted plane, in mm (read only). |
| **Ground Truth** | Check it and type the measured distance to the wall in mm. This turns on the **Z Accuracy** metric. While it is unchecked, the field follows the measured distance, so checking it starts from the current reading. |
| **Limit Captures** | Check it and enter a number (minimum 1, default 1) to stop recording after that many captures. |
| **Angle** | Angle between the camera axis and the plane normal, in degrees (read only). |

The panel also shows the usual depth sensor controls from the RealSense Viewer, including loading a JSON preset (see [[D400 Series Visual Presets]]). The *Enable Max Usable Range* and *Enable IR Reflectivity* options are hidden from the option list. When you load a preset, the tool restarts the streams with the same profiles.

## Reading the metrics

The **Metrics** section lists each metric with its latest value and unit. Expand a metric to see its description and a plot of its last 200 values. The tool calculates the metrics in a background thread and waits 80 ms after each frame, so values update at a readable rate.

How the metrics are calculated:

1. Every pixel inside the ROI with a non-zero depth value counts as a valid point. Each point is deprojected to 3D with the depth intrinsics.
2. A plane is fitted to the 3D points (a fit through the centroid of the points; see [`depth-metrics.h`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/depth-metrics.h)). At least 3 valid points are needed.
3. **Distance** is the distance from the camera to the plane along its normal (mm). **Angle** is `acos(|c|)` in degrees, where `c` is the Z component of the plane normal.
4. Fill rate is calculated from all valid points. The other metrics run only once a plane fit is found. Before calculating Z accuracy, sub-pixel RMS and plane fit RMS, the tool drops the closest 0.5% and the farthest 0.5% of the points (by Z).

The diagram from the tool's readme shows the terms used below:

![Depth pixel Zi and its projection ZPi on the fitted plane](https://raw.githubusercontent.com/realsenseai/librealsense/development/tools/depth-quality/res/Zi_ZPi.png)

### Fill-Rate

Percentage of pixels in the ROI with a valid (non-zero) depth value:

`Fill-Rate = 100 × valid pixels / (ROI width × ROI height)`

### Plane Fit RMS Error

Spatial noise: the RMS of the distances from each point to the fitted plane, shown as a percentage of the distance to the plane:

`RMS (mm) = sqrt( Σ distᵢ² / n )`, `Plane Fit RMS Error (%) = 100 × RMS / Distance`

The recorded CSV has both the percentage and the value in mm.

### Subpixel RMS Error

Sub-pixel accuracy, in pixels. For each point the tool compares the disparity of the measured point with the disparity of its projection onto the plane:

`Dᵢ = BL × FL / Zᵢ`, `DPᵢ = BL × FL / ZPᵢ`, `RMS = sqrt( Σ (Dᵢ − DPᵢ)² / n )`

- `BL`: stereo baseline. The tool reads it from the extrinsics between infrared stream 1 and infrared stream 2 of the depth sensor. If they are not available it shows `Extrinsic parameters are not available`.
- `FL`: focal length `fx` of the depth stream, in pixels.
- `Zᵢ`, `ZPᵢ`: in the code, the distance from the camera to the measured point and to its projection on the plane.

### Z Accuracy

Shown only while **Ground Truth** is checked and its value is greater than 0. Result in percent of the ground-truth distance (GT):

1. The tool finds where the ray through the centre of the frame meets the fitted plane, and takes the offset between that point's Z and GT (mm).
2. For each point, the Z error is that offset plus the point's signed distance to the fitted plane (mm).
3. `Z Accuracy (%) = 100 × median(Z errors) / GT`

A positive value means the fitted plane is farther than the ground truth (overshoot). A negative value means it is closer (undershoot).

### Temporal Noise

Depth noise over time, in percent, calculated over a sliding window of frames:

1. The tool keeps a FIFO of recent depth frames for the ROI (`NUM_IMAGES` is 40 in the code). Once 40 frames have been added, it drops the oldest one and calculates over the remaining frames.
2. For each pixel, it discards zero values and calculates the standard deviation and the median of the pixel's depth (mm) over the frames.
3. For each pixel it divides the standard deviation by the median. Pixels without data get 0.
4. `Temporal Noise (%) = 100 × median of all the per-pixel values`

Temporal noise appears only after the window has filled, and only while a plane fit is found.

## Recording metrics and frames

The save button at the bottom of the **Metrics** section records metrics and frames:

1. Optional: check **Limit Captures** and set how many captures to record.
2. Click the save button and choose a base file name in the dialog. Recording starts.
3. Click the button again to stop, or let the tool stop when the capture limit is reached. The tool then shows `Finished to record frames and matrics data`.

For each frameset recorded, the tool writes these files (`<base>` is the chosen name without its extension, `<n>` is the frame number):

| File | Content |
|---|---|
| `<base>_Depth_<n>.png` | Colorized depth frame |
| `<base>_Depth_<n>.raw`, `<base>_Infrared_<n>.raw` | Raw frame data of the depth and infrared streams |
| `<base>_Depth_<n>_metadata.csv`, `<base>_Infrared_<n>_metadata.csv` | Frame attributes and metadata (see [[Frame Metadata]]) |
| `<base>_<n>_3d_mesh.ply` | Point cloud of the 3D view, written only when a texture source is selected |

When recording stops, the tool also writes:

| File | Content |
|---|---|
| `<chosen name>_depth_metrics.csv` | Metrics for every captured frame (see below) |
| `<chosen name>_configuration.json` | Advanced-mode settings of the device. Written only when you stop recording with the button and the device supports advanced mode. |

The metrics CSV contains:

- **Device Info**: name, product ID, serial number and firmware version, and the active streaming profiles (stream, format, resolution, FPS).
- **Environment**: `Plane-Fit_distance_mm` and `Ground-Truth_Distance_mm` (`N/A` when no plane fit is found or ground truth is not used).
- One row per sample with the columns `Sample Id`, `Frame #`, `Timestamp (ms)`, then one column per metric: `Fill-Rate %`, `Z Accuracy %`, `Plane Fit RMS Error %`, `Subpixel RMS Error pixel`, `Temporal Noise %` and `Plane Fit RMS Error mm`. A cell stays empty when the metric was not calculated for that frame.

To record a standard `.bag` file instead, use the RealSense Viewer or `rs-record` (see [[Record and Playback]]).

## Metrics reference

The metrics are defined in [`rs-depth-quality.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/rs-depth-quality.cpp), in this order. *Plot range* is the vertical range of the plot shown when you expand a metric. *Needs plane fit* means the metric is hidden while no plane is found.

<!-- wikigen:table-metrics -->
| Metric | Unit | Plot range | Needs plane fit | Calculation |
|---|---|---|---|---|
| `Fill-Rate` | % | 0 – 100 | No | Valid pixels / ROI pixels × 100 |
| `Z Accuracy` | % | -10 – 10 | Yes | 100 × median(Z error to ground truth) / GT; only while ground truth is set |
| `Plane Fit RMS Error` | % | 0 – 5 | Yes | RMS of point-to-plane distances, as % of the distance to the plane (CSV also has mm) |
| `Subpixel RMS Error` | pixel | 0 – 1 | Yes | RMS of the disparity difference between each point and its plane projection |
| `Temporal Noise` | % | 0 – 100 | Yes | 100 × median over the ROI of per-pixel (standard deviation / median) over a sliding frame window |
<!-- /wikigen:table-metrics -->

Fixed values used by the tool ([`depth-quality-model.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/depth-quality-model.cpp), [`rs-depth-quality.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/rs-depth-quality.cpp)):

| Setting | Value |
|---|---|
| Default ROI | 40% of width and height, centred |
| Distance guide range | 300 – 2000 mm |
| Angle that triggers rotate messages | above 10 degrees |
| Outliers removed before plane-fit metrics | 0.5% closest and 0.5% farthest points |
| Temporal noise window (`NUM_IMAGES`) | 40 frames |
| Values kept per metric plot | 200 |
| Delay after each processed frame | 80 ms |

## Command-line options

The tool uses the shared command-line parser [`common/cli.h`](https://github.com/realsenseai/librealsense/blob/development/common/cli.h) (`rs2::cli`) and adds no options of its own:

<!-- wikigen:table-cli -->
| Option | Description |
|---|---|
| `--debug` | Turn on librealsense debug logs. Only available when built with `BUILD_EASYLOGGINGPP` (default `ON`). |
| `--eth` | Detect DDS devices, even if disabled in the configuration. Ignored when not built with DDS support. |
| `--eth-only` | Use only DDS devices; do not look for USB/MIPI devices. Ignored when not built with DDS support. |
| `--no-eth` | Do not detect DDS devices, even if enabled in the configuration. No-op when not built with DDS support. |
| `--domain-id <0-232>` | Domain ID to use with DDS devices (default 0). Ignored when not built with DDS support. |
| `--`, `--ignore_rest` | Ignore the rest of the labeled arguments following this flag. |
| `--version` | Display version information and exit. |
| `-h`, `--help` | Display usage information and exit. |
<!-- /wikigen:table-cli -->

DDS support is the CMake option `BUILD_WITH_DDS` (default `OFF`). When it is on, `--eth`, `--eth-only` and `--no-eth` cannot be combined, and `--domain-id` cannot be combined with `--no-eth`. See [[Ethernet and DDS Cameras]].

## Source and related pages

- Source: [`tools/depth-quality`](https://github.com/realsenseai/librealsense/tree/development/tools/depth-quality)
  - [`rs-depth-quality.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/rs-depth-quality.cpp): command line, metric definitions and calculations
  - [`depth-quality-model.cpp`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/depth-quality-model.cpp) and [`depth-quality-model.h`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/depth-quality-model.h): UI, guidance and recording
  - [`depth-metrics.h`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/depth-metrics.h): plane fit, distance and angle
  - [`readme.md`](https://github.com/realsenseai/librealsense/blob/development/tools/depth-quality/readme.md): metric diagrams and formulas
- [[Realsense Viewer Manual]]: the Viewer shares the UI and sensor controls used by this tool
- [[Command Line Tools]]
- [[D400 Series Visual Presets]] and [[D400 Advanced Mode]]
- [[Calibration]]
- [[Post Processing Filters]]

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
