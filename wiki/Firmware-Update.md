This page explains how to update, recover and back up the firmware of a RealSense camera with the SDK. It is for anyone who needs to flash a new firmware image, either from the command line with `rs-fw-update` or from the [[Realsense Viewer Manual]], and for developers who want to call the update API directly.

## Before you start

- A **signed** firmware image (`.bin`) is required. The SDK does not ship a bundled firmware binary; download the image for your device from the [RealSense firmware releases](https://dev.realsenseai.com/docs/firmware-updates) page.
- For D400 cameras, the [D400 firmware releases](https://dev.realsenseai.com/docs/firmware-releases-d400) page packs the image in a zip file; the file to use is the one named like `Signed_Image_UVC_<firmware_version>.bin`.
- Do not run other applications that use the camera (the Viewer, for example) while `rs-fw-update` runs.
- Do not disconnect the camera during the update. The tool prints a warning if the camera is connected through a USB 2 port; if the update fails, move the camera to a USB 3 port and try again.
- `rs-fw-update` is built with the other tools: it is part of the `BUILD_TOOLS` option (see [[Build Configuration]]).

## Update with rs-fw-update

List the connected devices:

```
rs-fw-update -l
```

Each line shows the device description, its *update serial number*, its firmware version and, for devices that report it, the SMCU firmware version:

```
Connected devices:
1) [USB] Intel RealSense D435IF s/n 038322070306, update serial number: 039223050231, firmware version: 5.15.1
```

If one device is connected, pass only the image:

```
rs-fw-update -f Signed_Image_UVC_5_15_1_0.bin
```

If several devices are connected, `-s` with the serial number is mandatory:

```
rs-fw-update -s 038322070306 -f Signed_Image_UVC_5_15_1_0.bin
```

The tool checks that the image is compatible with the device, switches the device to update state, writes the image, then waits (up to 15 seconds) for the device to reconnect and prints the new firmware version. When the output is a terminal, a progress percentage is shown.

If the image is not compatible with the camera the tool prints `This firmware version is not compatible with <camera name>` and exits with a failure code.

## Recover a camera

A camera that failed an update can enumerate in recovery mode. In the device list it appears as a recovery device, for example `Intel RealSense D4XX Recovery`. Use `-r` together with `-f`:

```
rs-fw-update -r -f Signed_Image_UVC_5_15_1_0.bin
```

- If more than one recovery device is connected, add `-s` with the device's update serial number.
- For a GMSL MIPI recovery device (D457) the tool does not wait for the device to reappear. It prints `For GMSL MIPI device please reboot, or reload d4xx driver` followed by `sudo rmmod d4xx && sudo modprobe d4xx`, or you can reboot the system.
- Without `-r`, a single connected device in recovery mode makes the tool stop with `Device is in recovery mode, use -r to recover`.

## Back up the flash

`-b` saves a copy of the camera flash to the given path before the update (or on its own):

```
rs-fw-update -s 038322070306 -b flash_backup.bin
```

If the device does not support flash backup the tool prints `Backup flash is not supported`.

## Unsigned firmware

`-u` together with `-f` calls the unsigned-update path. It is available only for unlocked cameras.

## Command-line flags

| Flag | Description |
|---|---|
| `-l`, `--list_devices` | List all available devices |
| `-f`, `--file <path>` | Path of the firmware image file |
| `-s`, `--serial_number <string>` | Serial number of the device to update; mandatory if more than one device is connected |
| `-r`, `--recover` | Recover connected devices which are in recovery mode |
| `-u`, `--unsigned` | Update unsigned firmware, available only for unlocked cameras |
| `-b`, `--backup <path>` | Create a backup of the camera flash and save it to the given path |
| `-h` | Show usage information |

Run without arguments, the tool prints `Nothing to do` and lists the devices. The default log level is warning; the shared tool options (such as `--debug`) are described in [[Command Line Tools]].

## Update from the RealSense Viewer

The Viewer offers an **Update Firmware** entry for a connected device, and can also notify you when a newer recommended version is available (manually through *Check for updates*, or automatically when a device connects). See [[Realsense Viewer Manual]] and the repository's [Viewer SW/FW updates](https://github.com/realsenseai/librealsense/blob/development/doc/viewer-sw-fw-update.md) document.

## Use the update API

The same flow is available from C++ through `rs2::updatable` and `rs2::update_device` ([rs_device.hpp](https://github.com/realsenseai/librealsense/blob/development/include/librealsense2/hpp/rs_device.hpp)):

- `rs2::updatable::enter_update_state()` switches the device to update (DFU) state; a new `rs2::update_device` then appears.
- `rs2::update_device::update(fw_image, callback)` writes the image; the callback receives progress as a float.
- `rs2::updatable::create_flash_backup(callback)` returns the flash contents.
- `rs2::updatable::update_unsigned(image, callback)` is the unsigned path.
- `rs2::device::is_in_recovery_mode()` tells whether the device is in recovery mode.

The complete flow, including compatibility checks and waiting for the device to reconnect, is in [rs-fw-update.cpp](https://github.com/realsenseai/librealsense/blob/development/tools/fw-update/rs-fw-update.cpp). The tool's own README is [here](https://github.com/realsenseai/librealsense/blob/development/tools/fw-update/readme.md); the library side lives in [src/fw-update](https://github.com/realsenseai/librealsense/blob/development/src/fw-update).

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
