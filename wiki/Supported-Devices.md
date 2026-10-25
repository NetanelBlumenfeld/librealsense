This page lists the RealSense devices that the SDK recognises, taken from the product tables in the source, and explains where to find the platform and motion-sensor details. It is for anyone who wants to know whether a camera model is handled by the SDK. The tables are generated from [d400-private.h](https://github.com/realsenseai/librealsense/blob/development/src/ds/d400/d400-private.h) and [d500-private.h](https://github.com/realsenseai/librealsense/blob/development/src/ds/d500/d500-private.h).

## How the SDK identifies a device

The SDK reads the USB product ID (PID) of a device and creates the device class that belongs to it. The D400 factory (`src/ds/d400/d400-factory.cpp`) throws "Unsupported RS400 model" for a PID it does not know, and the D500 factory (`src/ds/d500/d500-factory.cpp`) throws "unsupported D500 PID". When the `partial-device-allowed` context setting is on, both create a device with partial capabilities instead of failing.

The SDK's recovery entries are devices in firmware-update (recovery) mode; see [[Firmware Update]].

## D400 series

The names are the strings in `rs400_sku_names`; "Listed as supported" is membership in `rs400_sku_pid`. The entries with product IDs `0xABCC`, `0xABCE` and `0xABCF` are marked GMSL in the source comments; see [[D401 GMSL Dual RGB]] for the D401.

<!-- wikigen:table-d400-products -->
| Product name (as reported by the SDK) | Product ID | Listed as supported |
|---|---|---|
| RealSense D400 | `0x0AD1` | yes |
| RealSense D410 | `0x0AD2` | yes |
| RealSense D415 | `0x0AD3` | yes |
| RealSense D430 | `0x0AD4` | yes |
| RealSense D430 with Tracking Module | `0x0AD5` | yes |
| RealSense USB2 | `0x0AD6` | yes |
| RealSense D4XX Recovery | `0x0ADB` | recovery mode |
| RealSense D4XX USB2 Recovery | `0x0ADC` | recovery mode |
| RealSense IMU | `0x0AF2` | yes |
| RealSense D420 | `0x0AF6` | yes |
| RealSense D421 | `0x1155` | yes |
| RealSense D420 with Tracking Module | `0x0AFE` | yes |
| RealSense D410 with Tracking Module | `0x0AFF` | yes |
| RealSense D400 with Tracking Module | `0x0B00` | yes |
| RealSense D430 with Tracking and RGB Modules | `0x0B01` | yes |
| RealSense D460 | `0x0B03` | yes |
| RealSense D435 | `0x0B07` | yes |
| RealSense D436 | `0x1156` | yes |
| RealSense DS5U | `0x0B0C` | yes |
| RealSense D435I | `0x0B3A` | yes |
| RealSense F416 | `0x0B49` | yes |
| RealSense D430I | `0x0B4B` | yes |
| RealSense F416 with RGB Module | `0x0B52` | yes |
| RealSense D405 | `0x0B5B` | yes |
| RealSense D455 | `0x0B5C` | yes |
| RealSense D457 | `0xABCD` | yes |
| RealSense D4XX MIPI Recovery | `0xBBCD` | recovery mode |
| RealSense D430 | `0xABCE` | yes |
| RealSense D415 | `0xABCF` | yes |
| RealSense D401 | `0xABCC` | yes |
<!-- /wikigen:table-d400-products -->

### Motion sensors on D400 devices

According to the lists in `d400-private.h`:

- Depth cameras with a HID motion sensor (accelerometer and gyroscope): the D435I, D436, D430I and D455 PIDs (`d400_hid_sensors_pid`).
- Devices that report a fisheye (tracking module) stream: the PIDs of the D400, D410, D420, D430 "with Tracking Module" and D430 "with Tracking and RGB Modules" entries (`d400_fisheye_pid`).

Motion data is streamed like any other sensor, using the `RS2_STREAM_GYRO` and `RS2_STREAM_ACCEL` streams. This sketch is adapted from [rs-motion](https://github.com/realsenseai/librealsense/blob/development/examples/motion/rs-motion.cpp):

```cpp
rs2::pipeline pipe;
rs2::config cfg;
cfg.enable_stream(RS2_STREAM_GYRO);
cfg.enable_stream(RS2_STREAM_ACCEL);
pipe.start(cfg);

rs2::frameset frames = pipe.wait_for_frames();
if (rs2::motion_frame accel = frames.first_or_default(RS2_STREAM_ACCEL))
{
    rs2_vector sample = accel.get_motion_data();
}
```

The IMU calibration tool is [rs-imu-calibration](https://github.com/realsenseai/librealsense/tree/development/tools/rs-imu-calibration); see [[Calibration]]. The [rs-data-collect](https://github.com/realsenseai/librealsense/tree/development/tools/data-collect) tool can record motion data; see [[Command Line Tools]].

## D500 series

The names are the strings in `rs500_sku_names`; "Listed as supported" is membership in `rs500_sku_pid`. The D500 devices report the product line "D500". The D535 and D585 PIDs include dual RGB variants ("Dual RGB") and the "F" variants whose left and right imagers have IR-only covers, per comments in `d500-private.h`. See [[D500 Perception Features]] for the D500 features.

<!-- wikigen:table-d500-products -->
| Product name (as reported by the SDK) | Product ID | Listed as supported |
|---|---|---|
| RealSense D555 | `0x0B56` | yes |
| RealSense D555 Recovery | `0x0ADE` | recovery mode |
| RealSense D585 | `0x0B6A` | yes |
| RealSense D585S | `0x0B6B` | yes |
| RealSense D585S Recovery | `0x0ADD` | recovery mode |
| RealSense D500 Recovery | `0x0CFD` | recovery mode |
| RealSense D500 Recovery | `0x0CFE` | recovery mode |
| RealSense D5xx MIPI Recovery | `0xBBDD` | recovery mode |
| RealSense D535 Dual RGB | `0x0C01` | yes |
| RealSense D535 | `0x0C02` | yes |
| RealSense D535F | `0x0C03` | yes |
| RealSense D585 Dual RGB | `0x0C04` | yes |
| RealSense D585 | `0x0C05` | yes |
| RealSense D585F | `0x0C06` | yes |
| RealSense D585 Proto Dual RGB | `0x0C07` | yes |
| RealSense D585 Prototype | `0x0C08` | yes |
<!-- /wikigen:table-d500-products -->

## Platforms and backends

The SDK talks to the cameras through different backends depending on the operating system; see the installation pages:

- [[Installation Linux]], [[Installation Windows]], [[Installation macOS]]
- [[Installation Jetson]], [[Installation Raspberry Pi]], [[Android]]
- [[Docker]]

The repository's [support matrix](https://github.com/realsenseai/librealsense/blob/development/doc/support-matrix.md) describes how releases are published: each release first appears on the [releases page](https://github.com/realsenseai/librealsense/releases) as a pre-release and is promoted to the latest release once its validation report is approved. Other platforms are tested on demand.

## See also

- [[Firmware Update]]
- [[Build Configuration]]
- [[Realsense Viewer Manual]]

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
