> [Wiki](Home) ▸ **Troubleshooting Q&A**


**[Camera not working\detected](#camera-not-workingdetected)**
- [How do I see which RealSense cameras are connected?](#q-how-do-i-see-which-realsense-cameras-are-connected)
- [I connected the camera to the USB port but it is not recognized](#q-i-connected-the-camera-to-the-usb-port-but-it-is-not-recognized)
- [I'm using a virtual machine and the camera is not working](#q-im-using-a-virtual-machine-and-the-camera-is-not-working)


**[Installation](#installation)**
- [`Fatal error: libudev.h:` when building from sources](#q-fatal-error-libudevh-when-building-from-sources)
- [How do I enable librealsense logs?](#q-how-do-i-enable-librealsense-logs)
- [GCC Internal Error](#q-gcc-internal-error)
- [I ran the udev rules script but Linux still get `Permission denied`](#q-i-ran-the-udev-rules-script-but-linux-still-get-permission-denied)
- [dmesg shows: `uvcvideo: module verification failed: signature and/or required key missing - tainting kernel`](#q-dmesg-shows-uvcvideo-module-verification-failed-signature-andor-required-key-missing---tainting-kernel)
- [`sudo modprobe uvcvideo` produces dmesg: `uvc kernel module is not loaded`](#q-sudo-modprobe-uvcvideo-produces-dmesg-uvc-kernel-module-is-not-loaded)
- [Kernel patch script fails with fatal error: `openssl/opensslv.h`](#q-kernel-patch-script-fails-with-fatal-error-opensslopensslvh)
- [The RealSense Viewer shows `Multiple realsense udev-rules were found!`](#q-the-realsense-viewer-shows-multiple-realsense-udev-rules-were-found)
- [Build fails on `fastrtps` / `fastcdr` (Fast-DDS) dependencies](#q-build-fails-on-fastrtps--fastcdr-fast-dds-dependencies)
- [Build fails on `curl` dependencies](#q-build-fails-on-curl-dependencies)

**[Git](#git)**
- [Git returns `access timeout` when I try to clone the repository](#q-git-returns-access-timeout-when-i-try-to-clone-the-repository)


**[Python](#python)**
  - [CMake shows an error when I try to build with Python bindings](#q-cmake-shows-an-error-when-i-try-to-build-with-python-bindings)

**[Calibration](#calibration)**
 - [D400 Series Calibration Tools](#d400-series-calibration-tools)

-----------------





## Camera not working\detected


#### Q: How do I see which RealSense cameras are connected?

**Any OS:** run the `rs-enumerate-devices` tool shipped with the SDK. It lists every detected device with its serial number, firmware version and USB type. `rs-enumerate-devices -s` prints a one-line summary (name, serial number, firmware version) per device; if no device is found the tool prints `No device detected. Is it plugged in?`.

**Linux:**

```bash
$ lsusb | grep -iE '8086|38e5'
```
> Older SKUs enumerate with vendor ID `8086`, newer SKUs with `38e5`.

**Windows:**

1. Click the `Win-Key`+`X` keys
2. Choose `Device Manager`
3. Look under `Cameras` (or `Imaging devices` on older Windows builds) for "RealSense" cameras





#### Q: I connected the camera to the USB port but it is not recognized

> ❗️ Make sure your camera is connected using USB 3.0 ❗️ 

If your camera is connected via USB 3.0, check to make sure your OS detects the camera ([how?](#q-how-do-i-see-which-realsense-cameras-are-connected)).
The camera will not work without USB 3.0 (or higher)



#### Q: I'm using a virtual machine and the camera is not working

Due to the USB 3.0 translation layer between native hardware and virtual machine, the librealsense team does not support installation in a VM. If you do choose to try it, we recommend using VMware Workstation Player, and not Oracle VirtualBox for proper emulation of the USB3 controller.






## Installation 

#### Q: `Fatal error: libudev.h:` when building from sources
Install the `libudev-dev` package:
`sudo apt-get install libudev-dev`

In the current CMake scripts `libudev` is optional on Linux: [`src/linux/find_udev.cmake`](https://github.com/realsenseai/librealsense/blob/development/src/linux/find_udev.cmake) looks for `libudev.h`, and when it is not found CMake prints `UDEV not found; using polling device-watcher!` and builds a polling device watcher instead. With `libudev-dev` installed the SDK uses udev events for device connect/disconnect detection.


#### Q: How do I enable librealsense logs?
To change the log level of the librealsense logger, set an environment variable named **LRS_LOG_LEVEL**
to the desired log level:

**Linux:**

```bash
$ export LRS_LOG_LEVEL="<Log Level>"
```

**Windows:**

```bash
> set LRS_LOG_LEVEL=<Log Level>
```

> `<Log Level>` can be either `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL` or `NONE` (case-insensitive). On Windows `cmd`, do not put quotes around the value: `set` keeps them as part of the value and the level is then not recognized.

When the variable holds a valid level, librealsense writes the log to a file named after the current date and time (`<date-time>.log`) in the working directory of the application (see [`src/log.h`](https://github.com/realsenseai/librealsense/blob/development/src/log.h)).

See also [Troubleshooting Tips and Tricks](https://github.com/realsenseai/librealsense/blob/development/doc/troubleshooting.md) for kernel-level tracing on Linux.


#### Q: GCC Internal Error

The gcc compiler issues the following error while compiling:

> gcc: internal compiler error

This might indicate that you do not have enough memory or swap space on your machine. Try closing memory consuming applications, and if you are running inside a VM increase available RAM to at least 2 GB.




#### Q: I ran the udev rules script but Linux still get `Permission denied`

First, try re-installing udev rules located in librealsense source directory:
    sudo cp config/99-realsense-libusb.rules /etc/udev/rules.d/
    sudo udevadm control --reload-rules && sudo udevadm trigger

or run `./scripts/setup_udev_rules.sh` from the librealsense source directory, which installs the same rules file and reloads the rules.

If the issue persists, the cause might be due to the fact that your user is not part of the `plugdev` group (the rules in `config/99-realsense-libusb.rules` set `GROUP:="plugdev"`).




#### Q: `dmesg` shows: `uvcvideo: module verification failed: signature and/or required key missing - tainting kernel`

This is a standard warning issued since Kernel 4.4-30+, it is only a notification and does not affect module's functionality.



#### Q: `sudo modprobe uvcvideo` produces `dmesg: uvc kernel module is not loaded`

This issue is caused since the patched module kernel version is incompatible with the resident kernel.
Verify the actual kernel version with `uname -r`.
Revert and proceed from the [Install dependencies](https://github.com/realsenseai/librealsense/blob/development/doc/installation.md#install-dependencies) step of the Linux installation guide.



#### Q: Kernel patch script fails with fatal error: `openssl/opensslv.h`

Running `./scripts/patch-realsense-ubuntu-lts-hwe.sh` (or the other `patch-realsense-*.sh` scripts) fails with `fatal error: openssl/opensslv.h`.
This is usually caused due to a missing dependency.
Install the *openssl* development package (`libssl-dev`) as described in the [Install dependencies](https://github.com/realsenseai/librealsense/blob/development/doc/installation.md#install-dependencies) step.



#### Q: The RealSense Viewer shows `Multiple realsense udev-rules were found!`

The RealSense Viewer on Linux prints this notification when both `/etc/udev/rules.d/99-realsense-libusb.rules` (manual installation from source) and `/lib/udev/rules.d/60-librealsense2-udev-rules.rules` (Debian package) exist. Remove the installation you do not need (manual or Debian package).



#### Q: Build fails on `fastrtps` / `fastcdr` (Fast-DDS) dependencies

The build tries to integrate the eProsima Fast-DDS library and cannot build or find it on the current machine. Fast-DDS is only built when `BUILD_WITH_DDS` is `ON` (default `OFF`); build with `-DBUILD_WITH_DDS=OFF`.



#### Q: Build fails on `curl` dependencies

libcurl is pulled into the build when any of `CHECK_FOR_UPDATES`, `ENABLE_AI_ASSISTANT` or `ENABLE_STATS` is `ON` (all default `ON` except on macOS, where `CHECK_FOR_UPDATES` and `ENABLE_AI_ASSISTANT` default to `OFF`). If libcurl cannot be built or found on the current machine, turn these off, e.g. `-DCHECK_FOR_UPDATES=OFF -DENABLE_AI_ASSISTANT=OFF -DENABLE_STATS=OFF`. See [[Build Configuration]].











## Git

#### Q: Git returns `access timeout` when I try to clone the repository

    git.launchpad... access timeout

This usually happens when your computer is behind a firewall, try to configure git to use the appropriate proxy server







## Python

#### Q: CMake shows an error when I try to build with Python bindings
The following message appears when configuring with `-DBUILD_PYTHON_BINDINGS=ON`:

>  Could NOT find PythonInterp (missing: PYTHON_EXECUTABLE)

(With CMake 3.12 or newer, [`CMake/external_pybind11.cmake`](https://github.com/realsenseai/librealsense/blob/development/CMake/external_pybind11.cmake) uses `find_package(Python ...)` instead of `PythonInterp`, so the message names `Python`.)

Please check the following:
- Make sure Python is installed
- **On Windows**: If you've just installed python, reboot (or log off) and try again
  - Try the answers suggested here: [building-opencv-libraries-from-source-files](https://stackoverflow.com/questions/9119253/building-opencv-libraries-from-source-files)


>   Python config failure: Python is 32-bit, chosen compiler is 64-bit

If this message appears you should install python 64 bit

## Calibration
#### D400 Series Calibration Tools
* On-Chip (self) calibration is available directly from the [RealSense Viewer](https://github.com/realsenseai/librealsense/wiki/Realsense-Viewer-Manual#on-chip--self-calibration) and via the SDK API. See [Self-Calibration for Depth Cameras](https://dev.realsenseai.com/docs/self-calibration-for-depth-cameras).
* For the full Dynamic Calibration tools please check the [D400 Series Calibration Tools User Guide](https://dev.realsenseai.com/docs/intel-realsensetm-d400-series-calibration-tools-user-guide).

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
