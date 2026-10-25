This page covers installing the RealSense SDK on Ubuntu, either from the pre-built Debian packages or by building the SDK from source. It is for anyone setting up a Linux machine to run the RealSense Viewer, the command-line tools or their own applications. The repository's own guides are [distribution_linux.md](https://github.com/realsenseai/librealsense/blob/development/doc/distribution_linux.md) (packages) and [installation.md](https://github.com/realsenseai/librealsense/blob/development/doc/installation.md) (build from source).

## Choose a method

| Method | Use it when |
|---|---|
| [Debian packages](#install-the-debian-packages) | You want the SDK, tools and kernel drivers installed and upgraded through `apt`. This is the recommended path. |
| [Build from source](#build-from-source) | You use a non-LTS Ubuntu kernel, need to integrate your own patches or modules, or need to adjust the patches for another kernel or distribution. |
| [RSUSB backend](#build-without-kernel-patches) | Kernel patching is not possible on your platform. |

The repository guides list Ubuntu 20, 22, 24 and 26 LTS as the supported versions. Virtual machines are not supported by the SDK team, because of the USB 3.0 translation layer between the hardware and the VM.

## Install the Debian packages

1. Register the server's public key:
   ```sh
   sudo mkdir -p /etc/apt/keyrings
   curl -sSf https://librealsense.realsenseai.com/Debian/librealsenseai.asc | \
   gpg --dearmor | sudo tee /etc/apt/keyrings/librealsenseai.gpg > /dev/null
   ```
2. Make sure apt HTTPS support is installed and add the repository:
   ```sh
   sudo apt-get install apt-transport-https
   echo "deb [signed-by=/etc/apt/keyrings/librealsenseai.gpg] https://librealsense.realsenseai.com/Debian/apt-repo `lsb_release -cs` main" | \
   sudo tee /etc/apt/sources.list.d/librealsense.list
   sudo apt-get update
   ```
3. Install the kernel drivers and the tools:
   ```sh
   sudo apt-get install librealsense2-dkms
   sudo apt-get install librealsense2-utils
   ```
4. Optionally install the developer and debug packages:
   ```sh
   sudo apt-get install librealsense2-dev
   sudo apt-get install librealsense2-dbg
   ```
5. Reconnect the camera and run `realsense-viewer` to verify the installation. `modinfo uvcvideo | grep "version:"` should include the string `realsense`.

### Upgrade and uninstall

- Upgrade everything: `sudo apt-get update && sudo apt-get upgrade`. To upgrade only some packages: `sudo apt-get --only-upgrade install librealsense2-utils librealsense2-dkms`.
- Remove one package: `sudo apt-get purge <package-name>`. A package can be removed only when no other installed package depends on it; for example `librealsense2-udev-rules` requires `librealsense2` to be removed first.
- Remove all RealSense packages: `dpkg -l | grep "realsense" | cut -d " " -f 3 | xargs sudo dpkg --purge`.

### Package contents

| Package | Content |
|---|---|
| `librealsense2-udev-rules` | Device permissions at the kernel level |
| `librealsense2-dkms` | DKMS package with the depth-camera kernel extensions |
| `librealsense2` | SDK runtime (.so) and configuration files |
| `librealsense2-utils` | Demos and tools |
| `librealsense2-dev` | Header files and symbolic link for developers |
| `librealsense2-dbg` | Debug symbols |
| `librealsense2-gl` | GLSL extension module runtime and configuration |
| `librealsense2-gl-dev` | GLSL development header files and symbolic link |
| `librealsense2-gl-dbg` | GLSL debug symbols |

For end-of-life Ubuntu releases the guide points to older SDK versions: Ubuntu 16 to [2.51.1](https://github.com/realsenseai/librealsense/releases/tag/v2.51.1) and Ubuntu 18 to [2.55.1](https://github.com/realsenseai/librealsense/releases/tag/v2.55.1).

## Build from source

### Install the dependencies

```sh
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install libusb-1.0-0-dev
sudo apt-get install libudev-dev
sudo apt-get install libssl-dev pkg-config libgtk-3-dev
sudo apt-get install git wget cmake build-essential
sudo apt-get install libglfw3-dev libgl1-mesa-dev libglu1-mesa-dev at
```

- Unplug any connected RealSense camera before installing the last group.
- The packages are installed in separate commands because some platforms hit dependency problems when they are installed in one step.
- `glfw3`, `mesa` and `gtk` are needed only for the OpenGL-enabled examples. The core library and many tools can be deployed on headless systems.
- `libudev-dev` is optional but recommended: with it the SDK detects USB devices through events; without it, it polls on a timer.
- Some CMake flags (for example CUDA) need CMake 3.8 or newer, which apt may not provide on Ubuntu LTS; get one from [cmake.org](https://cmake.org/download/).

### Get the code and set permissions

```sh
git clone https://github.com/realsenseai/librealsense.git
cd librealsense
./scripts/setup_udev_rules.sh
```

[`setup_udev_rules.sh`](https://github.com/realsenseai/librealsense/blob/development/scripts/setup_udev_rules.sh) needs `v4l2-ctl` (`sudo apt install v4l-utils`). It copies [`config/99-realsense-libusb.rules`](https://github.com/realsenseai/librealsense/blob/development/config/99-realsense-libusb.rules) to `/etc/udev/rules.d/` (and, when the script detects an IPU6 or Tegra platform, [`99-realsense-d4xx-mipi-dfu.rules`](https://github.com/realsenseai/librealsense/blob/development/config/99-realsense-d4xx-mipi-dfu.rules)) and reloads the udev rules. Remove the permissions again with `./scripts/setup_udev_rules.sh --uninstall`.

If it reports `Multiple realsense udev-rules were found!`, you have both the Debian packages and a manual installation; remove one of them.

### Patch the kernel modules (optional)

Most RealSense cameras work with the stock kernel, including per-frame metadata, which the guide states is part of the mainline `uvcvideo` driver. Patching is needed only for newly released camera models that the distribution's `uvcvideo` does not recognise yet; without it streaming works but per-frame metadata is not available.

- Ubuntu LTS kernels 5.15, 5.19, 6.5, 6.8, 6.11 and 6.14: `./scripts/patch-realsense-ubuntu-lts-hwe.sh`
- Ubuntu 20 with an LTS kernel older than 5.13: `./scripts/patch-realsense-ubuntu-lts.sh`

The script downloads, patches and builds the affected kernel modules, then tries to insert the patched module in place of the active one. If that fails, the original modules are restored. Check the result with `sudo dmesg | tail -n 50`; the log should show that a new `uvcvideo` driver was registered.

Running RealSense cameras on Linux with patched drivers requires a kernel that can be modified; some OEMs lock the kernel, and unlocking may need a BIOS setting change.

### Build the SDK

```sh
mkdir build && cd build
cmake ../ -DBUILD_EXAMPLES=true
sudo make uninstall && make clean && make && sudo make install
```

- The default build produces the core shared library and the unit tests in Debug mode; add `-DCMAKE_BUILD_TYPE=Release` for an optimised build.
- For systems without OpenGL or X11, build only the textual examples: `cmake ../ -DBUILD_EXAMPLES=true -DBUILD_GRAPHICAL_EXAMPLES=false`.
- Use `make -j$(($(nproc)-1)) install` to compile in parallel on multi-core CPUs.
- The shared object is installed in `/usr/local/lib`, the headers in `/usr/local/include`, and the demos, tutorials and tests in `/usr/local/bin`.
- The Linux build uses the V4L2 backend by default.
- A `gcc: internal compiler error` usually means the machine is out of memory or swap; close other applications or give a VM at least 2 GB of RAM.

All CMake options are listed on [[Build Configuration]].

### Build without kernel patches

If kernel patching is not possible, or the steps above fail, the guide offers the libuvc-based script [`scripts/libuvc_installation.sh`](https://github.com/realsenseai/librealsense/blob/development/scripts/libuvc_installation.sh), described in [libuvc_installation.md](https://github.com/realsenseai/librealsense/blob/development/doc/libuvc_installation.md). That method is not officially validated, and the script assumes Ubuntu 16 with a graphic subsystem. The CMake option `FORCE_RSUSB_BACKEND` (optional on Linux) selects the RSUSB backend; `FORCE_LIBUVC` is deprecated in its favour.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `git.launchpad... access timeout` | Behind a firewall | Configure a proxy server |
| `uvcvideo: module verification failed: signature and/or required key missing - tainting kernel` in `dmesg` | Standard warning | Informational only |
| `sudo modprobe uvcvideo` gives `uvc kernel module is not loaded` | The patched module does not match the running kernel | Check `uname -r`, revert, and repeat from the dependency step |
| `patch-realsense-ubuntu-lts-hwe.sh` fails with `fatal error: openssl/opensslv.h` | Missing dependency | Install the OpenSSL development package (`libssl-dev`, listed above) |
| Build fails on `fastrtps` / `fastcdr` | The SDK cannot find or build eProsima Fast-DDS | Configure with `-DBUILD_WITH_DDS=OFF` |
| Build fails on `curl` | The SDK cannot find or build libcurl | Configure with `-DCHECK_FOR_UPDATES=OFF` |

For other problems see [[Troubleshooting Q&A]].

## Next steps

Run `realsense-viewer` ([[Realsense Viewer Manual]]), check the camera with `rs-enumerate-devices` ([[Command Line Tools]]) and update the camera firmware if needed ([[Firmware Update]]).

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
