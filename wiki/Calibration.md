RealSense depth cameras ship with a factory calibration stored on the device. librealsense provides tools and APIs to improve or restore it: the `rs-imu-calibration` Python script recalibrates the IMU (accelerometer and gyroscope), and the `rs2::auto_calibrated_device` API runs depth self-calibration (on-chip, tare, focal length, UV-map) and reads, writes or resets calibration tables. This page is for users and integrators who want to run these calibrations from the command line or from their own code. To run the same calibrations from a GUI, see [[Realsense Viewer Manual]].

## Calibrate the IMU with rs-imu-calibration

[`tools/rs-imu-calibration/rs-imu-calibration.py`](https://github.com/realsenseai/librealsense/blob/development/tools/rs-imu-calibration/rs-imu-calibration.py) walks you through six camera orientations, computes accelerometer scale/alignment and bias plus gyroscope bias, and can write the result to the camera.

### Requirements

The script imports:

- `numpy`
- `pyrealsense2` (the librealsense Python wrapper)
- Python standard modules: `sys`, `json`, `ctypes`, `os`, `binascii`, `struct`, `time`, `enum`, `threading`, `csv`
- `select`, `tty`, `termios` on POSIX systems, `msvcrt` on Windows; on any other OS it raises `Unsupported OS`
- `pylab` (matplotlib) only when `-g` is given

The device must expose both a gyro and an accel stream in `motion_xyz32f` format; otherwise the script prints `No IMU sensor found.` or `Not all IMU streams found.`.

### Command-line arguments

The script parses `sys.argv` directly:

| Argument | Description |
|---|---|
| `-h`, `--help`, `/?` | Print usage and exit. |
| `-i <accel_file> [gyro_file]` | Compute the calibration from previously saved files instead of the interactive procedure. The gyro file is taken only if the next argument does not start with `-`. |
| `-s <serial_no>` | Calibrate the device with this serial number. Without it, the first device found is used. |
| `-g` | Show a graph of the accelerometer norm values: raw in blue, corrected in green. |

### Interactive procedure

```
python rs-imu-calibration.py
```

1. The script waits for a device, starts the IMU streams and, if the sensor supports `enable_motion_correction`, sets it to 0 so the raw (uncorrected) IMU values are used.
2. For each of the six orientations it prints `Align to direction:` followed by the target gravity vector and a label:
   - Mounting screw pointing down, device facing out
   - Mounting screw pointing left, device facing out
   - Mounting screw pointing up, device facing out
   - Mounting screw pointing right, device facing out
   - Viewing direction facing down
   - Viewing direction facing up
3. In the `rotate` state, bring the printed difference to zero. When the accelerometer reading is close to the target, the script waits 3 seconds for the camera to stay still (`wait_to_stable`), then collects 1000 accelerometer samples (`collect_data`). Samples away from the target are not collected; if the camera moves, it prints `WARNING: MOVING` and goes back to `rotate`.
4. Press **ESC** to abort.

When all six orientations are done:

- The script asks for a file suffix to save the raw data as `accel_<footer>.txt` and `gyro_<footer>.txt`. Press Enter to skip. Saved files can be used later with `-i`.
- It prints the least-squares solution, `norm (raw data  )` and `norm (fixed data)` (a good calibration is near 9.80665), and the per-axis accelerometer and gyroscope noise variances.
- It writes `calibration.json` and `calibration.bin` to the directory of the accel file, or to the current directory.
- It asks `Would you like to write the results to the camera? (Y/N)`. Any answer containing `Y` or `y` writes the IMU calibration table to the device through the debug protocol. The script then prints `SUCCESS: saved calibration to camera.` or `FAILED: failed to save calibration to camera.`.

### Calibration from saved files

```
python rs-imu-calibration.py -i accel_1.txt gyro_1.txt
```

The accel file is read as CSV rows `timestamp,x,y,z`. Each row is assigned to the orientation it is close to. The gyro bias is the mean of the gyro samples from the first 4000 timestamp units of the gyro file. A connected device is still required: the script waits for one before it reads the files.

### Output files

`calibration.json` contains `device_type` (always `"D435i"`) and an `imus` list with `accelerometer` and `gyroscope` objects. Each object has `scale_and_alignment`, `bias` and, when available, `noise_variances`. The gyroscope `scale_and_alignment` is always the identity matrix; the script computes only the gyro bias.

`calibration.bin` contains the generated IMU calibration table.

See also the tool's [README](https://github.com/realsenseai/librealsense/blob/development/tools/rs-imu-calibration/README.md) for photos of a holder setup.

## Run depth self-calibration from code

All depth self-calibrations go through `rs2::auto_calibrated_device` ([`rs_device.hpp`](https://github.com/realsenseai/librealsense/blob/development/include/librealsense2/hpp/rs_device.hpp)), C API in [`rs_device.h`](https://github.com/realsenseai/librealsense/blob/development/include/librealsense2/h/rs_device.h). A device supports it when it extends to `RS2_EXTENSION_AUTO_CALIBRATED_DEVICE`. In the library this interface is implemented by D400 devices ([`d400-auto-calibration.cpp`](https://github.com/realsenseai/librealsense/blob/development/src/ds/d400/d400-auto-calibration.cpp)), D500 devices ([`d500-auto-calibration.cpp`](https://github.com/realsenseai/librealsense/blob/development/src/ds/d500/d500-auto-calibration.cpp)), and DDS devices whose product line is `D500` (through [`auto-calibrated-proxy.cpp`](https://github.com/realsenseai/librealsense/blob/development/src/auto-calibrated-proxy.cpp)). See [[Supported Devices]] for the device list.

```cpp
rs2::device dev = profile.get_device();
if( dev.is< rs2::auto_calibrated_device >() )
{
    auto cal_dev = dev.as< rs2::auto_calibrated_device >();
    // ...
}
```

The run functions return the new calibration table as an `rs2::calibration_table` (`std::vector<uint8_t>`). As the on-chip-calib example notes in its comments, the device then uses the results, but they are not saved and are lost on reset or power cycle. To keep them, call `set_calibration_table()` and `write_calibration()` (see [Write and restore calibration tables](#write-and-restore-calibration-tables)). On failure the functions throw. The firmware error messages come from `handle_calibration_error()`, for example `Not enough depth pixels! - low fill factor)` or `Calibration failed to converge`. A timeout throws `Operation timed-out!`.

D400 on-chip and tare runs, D500 tare runs and D555 on-chip runs turn the depth sensor's thermal compensation option off while they run, if it is on, and turn it back on afterwards (`thermal_compensation_guard` in [`ds-calib-common.cpp`](https://github.com/realsenseai/librealsense/blob/development/src/ds/ds-calib-common.cpp)).

### On-chip calibration (OCC)

On-chip calibration needs no target. Start depth streaming, then call `run_on_chip_calibration()` with a JSON string of parameters (see [On-chip calibration JSON](#on-chip-calibration-json-d400)). An empty string uses the defaults.

```cpp
float health = 0;
rs2::calibration_table table = cal_dev.run_on_chip_calibration(
    R"({ "calib type": 0, "speed": 2, "scan parameter": 0 })", &health,
    []( const float progress ) { std::cout << progress << "%\n"; },
    9000 );
```

`"calib type"` selects the D400 algorithm: 0 for regular (depth noise), 1 for focal length, 2 for regular followed by focal length. See [Health-check values](#health-check-values) for how to read `health`.

On D500 devices the JSON content selects an action rather than parameters, and the flow depends on the device. See [D500 on-chip calibration](#d500-on-chip-calibration).

### Tare calibration

Tare calibration adjusts the absolute depth to a flat target at a known distance. Pass the ground truth in millimetres and a two-element `health` array. The function writes the health values before and after calibration to `health[0]` and `health[1]`.

```cpp
float health[2] = { -1.f, -1.f };
rs2::calibration_table table = cal_dev.run_tare_calibration(
    ground_truth_mm, R"({ "average step count": 20, "step count": 20, "accuracy": 2 })", health,
    []( const float progress ) { std::cout << progress << "%\n"; },
    5000 );
```

By default (`"apply preset": 1`) the run switches the visual preset to High Accuracy and restores the previous preset when it ends:

- On D400, this requires Advanced Mode. Otherwise the call throws `Camera "Advanced Mode" must be enabled before running this calibration`. See [[D400 Series Visual Presets]].
- On D500, if the current preset is Custom, the current controls are saved and restored afterwards *(new in the next release)*. This needs advanced-mode support; without it the call throws `Calibration with custom visual preset requires advanced mode support`. At v2.59.1 a tare run on D500 with the Custom preset threw `Calibration with custom visual preset is not supported`.

### Host-assisted calibration (D400)

On D400, both OCC and tare accept `"host assistance": 1`. In this mode the first call only starts the process and returns an empty table. You then pass depth frames to `process_calibration_frame()` until it returns a non-empty table:

```cpp
rs2::calibration_table table = cal_dev.run_on_chip_calibration( R"({ "host assistance": 1 })", &health, 5000 );
while( table.empty() )
{
    rs2::frameset fs = pipe.wait_for_frames();
    table = cal_dev.process_calibration_frame( fs.get_depth_frame(), &health, 5000 );
}
```

The library builds the follow-up firmware calls itself: the fill factors, and `"depth"` for tare. Frames must carry `RS2_FRAME_METADATA_FRAME_COUNTER` and `RS2_FRAME_METADATA_FRAME_TIMESTAMP` metadata. Otherwise the call throws `missing FRAME_COUNTER` or `missing FRAME_TIMESTAMP`. Frames that arrive out of order throw `Frames arrived in a wrong order!`. On D500, `process_calibration_frame()` throws `Process Calibration Frame not applicable for this device`. For a full Python flow, see [`depth_ucal_example.py`](https://github.com/realsenseai/librealsense/blob/development/wrappers/python/examples/depth_ucal_example.py).

### Target-based focal-length calibration

`run_focal_length_calibration()` takes two `rs2::frame_queue`s of left and right IR frames that show the calibration target, plus the target rectangle size in mm. It returns a calibration table with corrected focal lengths, and writes the corrected ratio (percent) and the target tilt angle (degrees) to `ratio` and `angle`. With `adjust_both_sides = 1` the correction is split between the left and right cameras. With 0 only the right camera is adjusted. The C API rejects other values. The header documents 1280x720 IR frames with the target in the centre 320x240 ROI.

```cpp
float ratio = 0, angle = 0;
auto table = cal_dev.run_focal_length_calibration( left_q, right_q, target_w_mm, target_h_mm,
                                                   1 /*adjust_both_sides*/, &ratio, &angle );
```

### UV-map calibration and target distance

- `run_uv_map_calibration()` calibrates the depth-to-RGB mapping from left IR, color and depth frame queues and returns up to four health numbers (px, py, fx, fy). The C API accepts only `py_px_only` values 0 or 1. On D500 it throws `UV Map Calibration not applicable for this device`.
- `calculate_target_z()` returns the distance in mm from the camera to the calibration target plane. It uses the first queue (left IR on D400) and needs at least 10 frames. You can use the result as the ground truth for tare calibration.

The Viewer uses these functions in [`common/on-chip-calib.cpp`](https://github.com/realsenseai/librealsense/blob/development/common/on-chip-calib.cpp).

### D500 on-chip calibration

On D500 devices `run_on_chip_calibration()` picks one of three flows ([`d500-auto-calibration.cpp`](https://github.com/realsenseai/librealsense/blob/development/src/ds/d500/d500-auto-calibration.cpp)):

- **Devices whose name contains `D555`**: classic OCC. Only the JSON keys `"speed"` (default 3, range 0-4), `"scan parameter"` (default 0, range 0-1) and `"data sampling"` (default 1, range 0-1) are read.
- **Devices that use interactive triggered calibration**: the D5x5 family PIDs and the D585S (`uses_interactive_triggered_calibration()` in [`d500-private.h`](https://github.com/realsenseai/librealsense/blob/development/src/ds/d500/d500-private.h)). The JSON must contain one of the action tokens below. `"calib run"` stops at the health-check state with the candidate table in RAM. Then call again with `"calib try new"` / `"calib try old"` to preview, `"calib commit"` to flash, or `"calib cancel"` to discard. `health` returns the `rect_health` value in pixels; the pass threshold in [`calibration-engine-interface.h`](https://github.com/realsenseai/librealsense/blob/development/src/calibration-engine-interface.h) is `< 0.4`. If the firmware does not report success, `health` is `-1`.
- **Other D500 devices**: triggered calibration with `"calib run"`, `"calib dry run"` or `"calib abort"`/`"calib cancel"`.

Action tokens recognised by `get_mode_from_json()`. The code searches for each token as a substring of the JSON text, in this order:

| Token | Mode |
|---|---|
| `calib dry run` | Dry run |
| `calib commit` | Commit |
| `calib cancel` | Abort |
| `calib try new` | Try (new table) |
| `calib try old` | Try (old table) |
| `calib run` | Run |
| `calib abort` | Abort (legacy alias) |

Any other content throws `run_on_chip_calibration called with wrong content in json file`. Python examples: [`d500_triggered_calibration.py`](https://github.com/realsenseai/librealsense/blob/development/wrappers/python/examples/d500_triggered_calibration.py), [`d5x5_interactive_triggered_calibration.py`](https://github.com/realsenseai/librealsense/blob/development/wrappers/python/examples/d5x5_interactive_triggered_calibration.py).

### Python

The Python wrapper exposes the same class as `pyrealsense2.auto_calibrated_device` ([`pyrs_device.cpp`](https://github.com/realsenseai/librealsense/blob/development/wrappers/python/pyrs_device.cpp)):

- `run_on_chip_calibration(json_content, [callback,] timeout_ms)` returns `(table, (health, 0.0))`.
- `run_tare_calibration(ground_truth_mm, json_content, [callback,] timeout_ms)` and `process_calibration_frame(frame, [callback,] timeout_ms)` return `(table, (health0, health1))`.

See [`depth_auto_calibration_example.py`](https://github.com/realsenseai/librealsense/blob/development/wrappers/python/examples/depth_auto_calibration_example.py).

## The on-chip-calib example

[`examples/on-chip-calib`](https://github.com/realsenseai/librealsense/tree/development/examples/on-chip-calib) (`rs-on-chip-calib`) shows the minimal OCC flow:

1. Start a pipeline with depth at 256x144, `RS2_FORMAT_Z16`, 90 fps, and wait for the first frames.
2. Build the JSON `{ "calib type": 0, "speed": 2, "scan parameter": 0 }`.
3. Get `rs2::auto_calibrated_device` from the device and call `run_on_chip_calibration( json, &health, callback, 9000 )`. Progress is printed as `progress = <n>%`.
4. Print `Completed successfully` and ask `Keep results? Yes/No`. On `y`/`yes` it calls `set_calibration_table( res )` and `write_calibration()` and prints `Results saved to flash`. Otherwise it prints `Results not saved`.

The example uses `rs2::cli` from [`common/cli.h`](https://github.com/realsenseai/librealsense/blob/development/common/cli.h), so it accepts:

| Flag | Description |
|---|---|
| `--debug` | Turn on librealsense debug logs |
| `--eth` | Detect DDS devices, even if disabled in the configuration (ignored without DDS support) |
| `--eth-only` | Use only DDS devices (ignored without DDS support) |
| `--no-eth` | Do not detect DDS devices (no-op without DDS support) |
| `--domain-id <0-232>` | DDS domain ID (ignored without DDS support) |
| `-h`, `--help` | Show usage |
| `--version` | Show version |

## Write and restore calibration tables

- `get_calibration_table()` reads the current depth calibration table from the device. Save it before calibrating if you may want to go back.
- `set_calibration_table( table )` makes the table active without saving it:
  - On D400, a depth coefficients table is loaded into the device RAM. An RGB calibration table is only stored for the next `write_calibration()`. Other table types throw.
  - On D500, the table must have the expected size for a depth or RGB table, and is sent to RAM.
- `write_calibration()` writes the table set by the last `set_calibration_table()` to flash. On D400, calling it before any table was set throws `Write calibration can be called only after set calibration table was called`.
- `reset_to_factory_calibration()` restores the factory calibration on the device.

Until you call `write_calibration()`, the new values are lost on reset or power cycle. To undo a calibration, either reset the camera, or set the saved table with `set_calibration_table()` and call `write_calibration()`.

```cpp
auto old_table = cal_dev.get_calibration_table();
auto new_table = cal_dev.run_on_chip_calibration( json, &health, 9000 );
if( keep )
{
    cal_dev.set_calibration_table( new_table );
    cal_dev.write_calibration();
}
else
    cal_dev.set_calibration_table( old_table );   // back to the previous values in RAM
```

`get_calibration_config()` and `set_calibration_config()` exchange a JSON calibration configuration. On D400 they throw `Calibration Config not applicable for this device`.

## API reference

### C++ (`include/librealsense2/hpp/rs_device.hpp`)

`typedef std::vector<uint8_t> calibration_table;`

`rs2::calibrated_device` (base of `auto_calibrated_device`):

<!-- wikigen:table-calibrated-device -->
| Signature | Description |
|---|---|
| `void write_calibration() const` | Write the table set by `set_calibration_table` to the device's flash. |
| `void reset_to_factory_calibration()` | Reset the device to factory calibration. |
<!-- /wikigen:table-calibrated-device -->

`rs2::auto_calibrated_device`:

<!-- wikigen:table-auto-calibrated-device -->
| Signature | Description |
|---|---|
| `template<class T> calibration_table run_on_chip_calibration(std::string json_content, float* health, T callback, int timeout_ms = 5000) const` | On-chip calibration with progress callback. |
| `calibration_table run_on_chip_calibration(std::string json_content, float* health, int timeout_ms = 5000) const` | On-chip calibration. |
| `template<class T> calibration_table run_tare_calibration(float ground_truth_mm, std::string json_content, float* health, T callback, int timeout_ms = 5000) const` | Tare calibration with progress callback. |
| `calibration_table run_tare_calibration(float ground_truth_mm, std::string json_content, float* health, int timeout_ms = 5000) const` | Tare calibration. |
| `template<class T> calibration_table process_calibration_frame(rs2::frame f, float* const health, T callback, int timeout_ms = 5000) const` | Feed a frame during host-assisted calibration, with progress callback. |
| `calibration_table process_calibration_frame(rs2::frame f, float* const health, int timeout_ms = 5000) const` | Feed a frame during host-assisted calibration. |
| `calibration_table get_calibration_table()` | Read the current calibration table. |
| `void set_calibration_table(const calibration_table& calibration)` | Set the current table (not saved until `write_calibration`). |
| `std::vector<uint8_t> run_focal_length_calibration(rs2::frame_queue left, rs2::frame_queue right, float target_w, float target_h, int adjust_both_sides, float* ratio, float* angle) const` | Target-based focal-length calibration. |
| `template<class T> std::vector<uint8_t> run_focal_length_calibration(rs2::frame_queue left, rs2::frame_queue right, float target_w, float target_h, int adjust_both_sides, float* ratio, float* angle, T callback) const` | Same, with progress callback. |
| `std::vector<uint8_t> run_uv_map_calibration(rs2::frame_queue left, rs2::frame_queue color, rs2::frame_queue depth, int py_px_only, float* health, int health_size) const` | Depth-RGB UV-map calibration. |
| `template<class T> std::vector<uint8_t> run_uv_map_calibration(rs2::frame_queue left, rs2::frame_queue color, rs2::frame_queue depth, int py_px_only, float* health, int health_size, T callback) const` | Same, with progress callback. |
| `float calculate_target_z(rs2::frame_queue queue1, rs2::frame_queue queue2, rs2::frame_queue queue3, float target_width, float target_height) const` | Distance to the target plane in mm. |
| `template<class T> float calculate_target_z(rs2::frame_queue queue1, rs2::frame_queue queue2, rs2::frame_queue queue3, float target_width, float target_height, T callback) const` | Same, with progress callback. |
| `std::string get_calibration_config() const` | Read the calibration config JSON. |
| `void set_calibration_config(const std::string& calibration_config_json_str) const` | Write the calibration config JSON. |
<!-- /wikigen:table-auto-calibrated-device -->

Related classes in the same header: `rs2::calibration_change_device` (`register_calibration_change_callback(T callback)`, callback receives `rs2_calibration_status`) and `rs2::device_calibration` (`trigger_device_calibration(rs2_calibration_type type)`).

### C (`include/librealsense2/h/rs_device.h`)

<!-- wikigen:table-c-api -->
| Function |
|---|
| `void rs2_reset_to_factory_calibration(const rs2_device* device, rs2_error** e);` |
| `void rs2_write_calibration(const rs2_device* device, rs2_error** e);` |
| `const rs2_raw_data_buffer* rs2_run_on_chip_calibration_cpp(rs2_device* device, const void* json_content, int content_size, float* health, rs2_update_progress_callback* progress_callback, int timeout_ms, rs2_error** error);` |
| `const rs2_raw_data_buffer* rs2_run_on_chip_calibration(rs2_device* device, const void* json_content, int content_size, float* health, rs2_update_progress_callback_ptr callback, void* client_data, int timeout_ms, rs2_error** error);` |
| `const rs2_raw_data_buffer* rs2_run_tare_calibration_cpp(rs2_device* dev, float ground_truth_mm, const void* json_content, int content_size, float* health, rs2_update_progress_callback* progress_callback, int timeout_ms, rs2_error** error);` |
| `const rs2_raw_data_buffer* rs2_process_calibration_frame(rs2_device* dev, const rs2_frame* f, float* const health, rs2_update_progress_callback* progress_callback, int timeout_ms, rs2_error** error);` |
| `const char* rs2_calibration_type_to_string( rs2_calibration_type );` |
| `const char* rs2_calibration_status_to_string( rs2_calibration_status );` |
| `void rs2_register_calibration_change_callback( rs2_device* dev, rs2_calibration_change_callback_ptr callback, void* user, rs2_error** error );` |
| `void rs2_register_calibration_change_callback_cpp( rs2_device* dev, rs2_calibration_change_callback* callback, rs2_error** error );` |
| `void rs2_trigger_device_calibration( rs2_device* dev, rs2_calibration_type type, rs2_error** error );` |
| `const rs2_raw_data_buffer* rs2_run_tare_calibration(rs2_device* dev, float ground_truth_mm, const void* json_content, int content_size, float* health, rs2_update_progress_callback_ptr callback, void* client_data, int timeout_ms, rs2_error** error);` |
| `const rs2_raw_data_buffer* rs2_get_calibration_table(const rs2_device* dev, rs2_error** error);` |
| `void rs2_set_calibration_table(const rs2_device* device, const void* calibration, int calibration_size, rs2_error** error);` |
| `const rs2_raw_data_buffer* rs2_run_focal_length_calibration_cpp(rs2_device* device, rs2_frame_queue* left_queue , rs2_frame_queue* right_queue, float target_width, float target_height, int adjust_both_sides, float* ratio, float* angle, rs2_update_progress_callback * progress_callback, rs2_error** error);` |
| `const rs2_raw_data_buffer* rs2_run_focal_length_calibration(rs2_device* device, rs2_frame_queue* left_queue, rs2_frame_queue* right_queue, float target_width, float target_height, int adjust_both_sides, float* ratio, float* angle, rs2_update_progress_callback_ptr callback, void* client_data, rs2_error** error);` |
| `const rs2_raw_data_buffer* rs2_run_uv_map_calibration_cpp(rs2_device* device, rs2_frame_queue* left_queue, rs2_frame_queue* color_queue, rs2_frame_queue* depth_queue, int py_px_only, float * health, int health_size, rs2_update_progress_callback * progress_callback, rs2_error** error);` |
| `const rs2_raw_data_buffer* rs2_run_uv_map_calibration(rs2_device* device, rs2_frame_queue* left_queue, rs2_frame_queue* color_queue, rs2_frame_queue* depth_queue, int py_px_only, float* health, int health_size, rs2_update_progress_callback_ptr callback, void* client_data, rs2_error** error);` |
| `float rs2_calculate_target_z_cpp(rs2_device* device, rs2_frame_queue* queue1, rs2_frame_queue* queue2, rs2_frame_queue* queue3, float target_width, float target_height, rs2_update_progress_callback* callback, rs2_error** error);` |
| `float rs2_calculate_target_z(rs2_device* device, rs2_frame_queue* queue1, rs2_frame_queue* queue2, rs2_frame_queue* queue3, float target_width, float target_height, rs2_update_progress_callback_ptr progress_callback, void* client_data, rs2_error** error);` |
| `const rs2_raw_data_buffer* rs2_get_calibration_config(rs2_device* device, rs2_error** error);` |
| `void rs2_set_calibration_config(rs2_device* device, const char* calibration_config_json_str,  rs2_error** error);` |
<!-- /wikigen:table-c-api -->

`rs2_run_on_chip_calibration*` requires a non-null `health`. A JSON pointer is checked only when `content_size > 0`. `rs2_write_calibration` and `rs2_reset_to_factory_calibration` throw `this device does not support auto calibration` and `this device does not support reset to factory calibration` on devices without the interface.

### Enums

<!-- wikigen:table-calibration-type -->
| `rs2_calibration_type` |
|---|
| `RS2_CALIBRATION_AUTO_DEPTH_TO_RGB` |
| `RS2_CALIBRATION_MANUAL_DEPTH_TO_RGB` |
| `RS2_CALIBRATION_THERMAL` |
<!-- /wikigen:table-calibration-type -->

<!-- wikigen:table-calibration-status -->
| `rs2_calibration_status` | Value | Header comment |
|---|---|---|
| `RS2_CALIBRATION_TRIGGERED` | 0 | AC triggered and is active; conditions are valid |
| `RS2_CALIBRATION_SPECIAL_FRAME` | 1 | Special frame received; expect a frame-drop! |
| `RS2_CALIBRATION_STARTED` | 2 | Have all frames in hand; starting processing |
| `RS2_CALIBRATION_NOT_NEEDED` | 3 | Finished; existing calibration within tolerances; nothing done! |
| `RS2_CALIBRATION_SUCCESSFUL` | 4 | Finished; have new calibration in-hand |
| `RS2_CALIBRATION_RETRY` | -1 | Initiating retry (asked for a new special frame) |
| `RS2_CALIBRATION_FAILED` | -2 | Unexpected: exception, device removed, stream stopped, etc. |
| `RS2_CALIBRATION_SCENE_INVALID` | -3 | Scene was not good enough for calibration; will retry |
| `RS2_CALIBRATION_BAD_RESULT` | -4 | Calibration finished, but results aren't good; will retry |
| `RS2_CALIBRATION_BAD_CONDITIONS` | -5 | Trigger was attempted but conditions (temp/APD) were invalid (still inactive) |
<!-- /wikigen:table-calibration-status -->

## JSON parameters

The JSON is parsed by `ds_calib_common::parse_json()` ([`ds-calib-common.cpp`](https://github.com/realsenseai/librealsense/blob/development/src/ds/ds-calib-common.cpp)) into key/integer pairs. Missing keys keep their defaults, and unknown keys are ignored. Keys contain spaces, as shown. Values outside the ranges listed below throw an `Auto calibration failed! Given value of '...' ... is out of range (...)` error. The defaults below come from the code and differ in places from the comments in `rs_device.h`.

### On-chip calibration JSON (D400)

From `auto_calibrated::run_on_chip_calibration()` in [`d400-auto-calibration.cpp`](https://github.com/realsenseai/librealsense/blob/development/src/ds/d400/d400-auto-calibration.cpp):

<!-- wikigen:table-occ-json -->
| Key | Default | Range / values | Used by |
|---|---|---|---|
| `calib type` | 0 | 0 = regular, 1 = focal length, 2 = regular + focal length | all |
| `speed` | 3 | 0 very fast, 1 fast, 2 medium, 3 slow, 4 white wall | type 0, 2 |
| `host assistance` | 0 | 0-3: 0 none, 1 start, 2 first feed, 3 second feed | all |
| `scan parameter` | 0 | 0 Py scan, 1 Rx scan | type 0, 2 |
| `data sampling` | 1 | 0 polling, 1 interrupt | type 0 (range-checked for type 2) |
| `fl step count` | 100 | 8-256 | type 1 |
| `fy scan range` | 40 | 1-60000 | type 1 |
| `keep new value after sucessful scan` | 1 | 0-1 | type 1, 2 |
| `fl data sampling` | 1 | 0-1 | type 1, 2 |
| `adjust both sides` | 0 | 0 right only, 1 both sides | type 1, 2 |
| `fl scan location` | 0 | 0-1 | type 1, 2 |
| `fy scan direction` | 0 | 0-1 | type 1, 2 |
| `white wall mode` | 0 | 0-1 | type 1, 2 |
| `scan only` | 0 | flag | all |
| `interactive scan` | 0 | flag | all |
| `step count v3` | 0 | number of `fill factor <i>` entries | host-assisted feed |
| `fill factor <i>` | 0 | per-step fill factor | host-assisted feed |
| `resize factor` | 5 | divisor of the frame size for the host-side ROI | host-assisted |
<!-- /wikigen:table-occ-json -->

`step count v3` and `fill factor <i>` are what the library itself sends during host-assisted OCC. You do not need to set them.

### Tare calibration JSON

From `run_tare_calibration()` in `d400-auto-calibration.cpp` and `d500-auto-calibration.cpp`:

<!-- wikigen:table-tare-json -->
| Key | Default | Range / values | Devices |
|---|---|---|---|
| `speed` | 3 | 0-4 | D400, D500 |
| `average step count` | 20 | 1-30 (frames to average) | D400, D500 |
| `step count` | 20 | 5-30 (max iteration steps) | D400, D500 |
| `accuracy` | 2 | 0 very high (0.025%), 1 high (0.05%), 2 medium (0.1%), 3 low (0.2%) | D400, D500 |
| `scan parameter` | 0 | 0 Py scan, 1 Rx scan | D400, D500 |
| `data sampling` | 0 | 0 polling, 1 interrupt | D400, D500 |
| `apply preset` | 1 | non-zero switches to the High Accuracy preset during the run | D400, D500 |
| `host assistance` | 0 | 0-3 | D400 |
| `depth` | 0 | 0 normal; > 0 feed host depth; -1 end feeding | D400 |
| `resize factor` | 5 | divisor of the frame size for the host-side ROI | D400 |
<!-- /wikigen:table-tare-json -->

The `rs_device.h` comment states a default of 10 for `step count` and writes one key as `average_step_count`. The code uses 20 and `average step count`.

## Health-check values

As documented in `rs_device.h`, the absolute value of the on-chip health check shows how far the current calibration is from the optimal one:

| Calibration | Good | Can be improved | Requires calibration |
|---|---|---|---|
| Regular (type 0) | [0, 0.25) | [0.25, 0.75) | [0.75, ...) |
| Focal length (type 1) | [0, 0.15) | [0.15, 0.75) | [0.75, ...) |

For `calib type` 2 on D400, the code packs both numbers into the single `health` float as an integer:

- bits 0-11: `round(|regular health| * 1000)`
- bits 12-23: `round(|focal-length health| * 1000)`
- bit 24: set if the regular health is negative
- bit 25: set if the focal-length health is negative

For tare, `health[0]` and `health[1]` are the firmware's health-check numbers before and after calibration. The library logs `(health + 1) * ground_truth_mm` as the Z computed from them. For UV-map calibration, `health` receives the px, py, fx, fy health numbers.

## Related

- [[Realsense Viewer Manual]]: on-chip, tare, focal-length and UV-map calibration from the Viewer UI ([`common/on-chip-calib.cpp`](https://github.com/realsenseai/librealsense/blob/development/common/on-chip-calib.cpp))
- [[D400 Series Visual Presets]]
- [[Supported Devices]]
- IMU calibration table parsing in the library: [`src/ds/ds-calib-parsers.cpp`](https://github.com/realsenseai/librealsense/blob/development/src/ds/ds-calib-parsers.cpp)

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
