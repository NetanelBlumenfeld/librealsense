This page explains how to get the RealSense SDK running on NVIDIA Jetson boards: from the Debian packages, from the `pyrealsense2` wheels, or by building the SDK with either the RSUSB or the native (kernel patched) backend. It is for anyone setting up a RealSense camera on a Jetson running NVIDIA's L4T Ubuntu image. The repository guide is [installation_jetson.md](https://github.com/realsenseai/librealsense/blob/development/doc/installation_jetson.md).

## Choose a backend

On Linux the SDK talks to the camera in one of two ways, and the choice is made when the SDK is configured with CMake:

| Backend | How it works | Trade-off |
|---|---|---|
| Native | Linux kernel drivers (Video4Linux for video, IIO for motion sensors). Needs kernel modules patched for RealSense. | Recommended, especially in production. |
| RSUSB | A user-space implementation of the UVC and HID protocols built into the SDK, selected with `-DFORCE_RSUSB_BACKEND=ON`. | No kernel patching, so it is quick for prototyping, with performance and functional limitations (the repository guide gives multi-camera as an example). |

`CMake/unix_config.cmake` selects `RS2_USE_LIBUVC_BACKEND` when `FORCE_RSUSB_BACKEND` is on and `RS2_USE_V4L2_BACKEND` otherwise. The two are mutually exclusive.

NVIDIA's L4T ships its own customised kernel, so the RealSense kernel modules have to be built against it. `scripts/patch-realsense-ubuntu-L4T.sh` does this without replacing the kernel image (see below).

## Install the Debian packages

1. Register the SDK's package server and key as described on [[Installation Linux]].
2. Install the tools and the developer package:
   ```sh
   sudo apt-get install librealsense2-utils
   sudo apt-get install librealsense2-dev
   ```
3. Reconnect the camera and run `realsense-viewer` to verify the installation.

With `librealsense2-dev` installed you can build an application with `g++ -std=c++11 filename.cpp -lrealsense2`. For a CMake project, adapt the [CMake example](https://github.com/realsenseai/librealsense/tree/development/examples/cmake).

The repository guide says the packages need JetPack 5.0.2 (L4T 35.1, CUDA 11.4) or later, because earlier versions have incompatible CUDA versions. It also notes that you may need to run the kernel patch script below for the camera to work on your platform.

## Install the Python wrapper

The `aarch64` wheels of [`pyrealsense2`](https://pypi.org/project/pyrealsense2/) are CUDA builds, one per JetPack line, and the wheel's Python version selects the JetPack it was built for (see [wrappers/python/readme.md](https://github.com/realsenseai/librealsense/blob/development/wrappers/python/readme.md)). Install with the Python version that matches your JetPack; the table is on the PyPI project page.

```sh
python3.<X> -m pip install pyrealsense2
```

For another Python version, or to run without CUDA, build the wrapper from source. See [[Python Wrapper]].

## Build with the RSUSB backend

This route avoids kernel patching. Run [scripts/libuvc_installation.sh](https://github.com/realsenseai/librealsense/blob/development/scripts/libuvc_installation.sh) from the repository. As written in the script it:

1. asks you to remove attached cameras if any `/dev/video*` device exists;
2. creates a swap file when no swap is active;
3. installs `git cmake libssl-dev freeglut3-dev libusb-1.0-0-dev pkg-config libgtk-3-dev unzip` with apt;
4. downloads the `master` source archive from GitHub and unpacks it;
5. copies `config/99-realsense-libusb.rules` and `config/99-realsense-d4xx-mipi-dfu.rules` to `/etc/udev/rules.d/` and reloads the rules;
6. configures with `cmake ../ -DFORCE_LIBUVC=true -DCMAKE_BUILD_TYPE=release`, then runs `make -j2` and `sudo make install`.

`FORCE_LIBUVC` is marked deprecated in `CMake/lrs_options.cmake` in favour of `FORCE_RSUSB_BACKEND`. If the CUDA development kit is installed, add `-DBUILD_WITH_CUDA=true` (default `OFF`) for better performance. See [[GPU Acceleration]].

## Build with the native backend

This route patches the kernel modules and builds the SDK against the V4L2 backend.

### Prepare

- Check that your board and JetPack version are supported by the script (table below).
- Make sure the board has an internet connection and enough free flash space (`df -h`).
- Disconnect attached RealSense cameras; the script asks you to do so when `/dev/video*` devices exist.

### Patch the kernel modules

From the repository root:

```sh
./scripts/patch-realsense-ubuntu-L4T.sh
```

The script reads the board model from `/proc/device-tree/model` and the L4T version from `/etc/nv_tegra_release`, and stops if the file is missing. It then:

1. shows the NVIDIA license for the kernel sources and waits for a key press (30 seconds);
2. installs `build-essential git libssl-dev curl`, and fetches the kernel sources for the matching tag with `scripts/Tegra/sync.sh`;
3. applies the RealSense patches and builds the `uvcvideo`, `v4l2-core`, `hid` and `iio` modules as needed for the version;
4. copies the modules: to your home folder for JetPack 4 and 5, or to `/lib/modules/<running kernel>/extra/` for JetPack 6 and later.

Supported L4T versions, as listed in the script:

| L4T version | JetPack | Kernel |
|---|---|---|
| 32.2.1, 32.2.3, 32.3.1, 32.4.3 | JetPack 4 releases up to 4.4 (patch set "4.4") | not set by the script |
| 32.4.4, 32.5, 32.5.1, 32.6.1, 32.7.1 | JetPack 4.4.1 line (32.7.1 is JetPack 4.6.1) | not set by the script |
| 35.1, 35.4.1, 35.6.4 | 5.0.2, 5.1.2, 5.1.6 | 5.10 |
| 36.3, 36.4, 36.4.3, 36.4.4, 36.4.7, 36.5 | 6.0, 6.1, 6.2, 6.2.1 (36.4.4 and 36.4.7), 6.2.2 (36.5) | 5.15 |
| 38.2, 38.2.1, 38.2.2, 38.4, 39.2 | 7.0 (38.2), 7.1 (38.4), 7.2 (39.2) | 6.8 |

Any other version makes the script stop with "Unsupported JetPack revision".

### Build the SDK

```sh
sudo apt-get install git libssl-dev libusb-1.0-0-dev libudev-dev pkg-config libgtk-3-dev -y
./scripts/setup_udev_rules.sh
mkdir build && cd build
cmake .. -DBUILD_EXAMPLES=true -DCMAKE_BUILD_TYPE=release -DFORCE_RSUSB_BACKEND=false -DBUILD_WITH_CUDA=true && make -j$(($(nproc)-1)) && sudo make install
```

`-DBUILD_WITH_CUDA=true` assumes the CUDA modules are installed; if they are not, use NVIDIA's SDK Manager to add them, or drop the flag. Finally connect the camera and run `realsense-viewer` ([[Realsense Viewer Manual]]).

## See also

- [[Installation Linux]]
- [[Build Configuration]]
- [[Supported Devices]]

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
