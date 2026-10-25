This page explains how to run the RealSense SDK inside a Docker container, either from a pre-built image or from an image you build with the scripts in the repository. It is for anyone who wants a reproducible environment for the SDK's command-line tools, Python bindings or their own applications. The files are in [scripts/Docker](https://github.com/realsenseai/librealsense/tree/development/scripts/Docker).

## What the image contains

According to the repository readme the image supports using RealSense devices, using the SDK API and running the basic examples. It does not include the graphical examples or IMU (motion) device support.

The [Dockerfile](https://github.com/realsenseai/librealsense/blob/development/scripts/Docker/Dockerfile) has two stages:

| Stage | Purpose |
|---|---|
| `librealsense-builder` | Starts from `ubuntu:20.04` (the `BASE_IMAGE` build argument), installs the build dependencies, downloads the source archive of release tag `v$LIBRS_VERSION` and builds it. |
| `librealsense` | The final image. It copies only the built binaries (`/opt/librealsense` is copied to `/usr/local/`) and the Python module, plus the runtime packages `libusb-1.0-0` and `udev`. |

The build configuration inside the builder stage is:

- `-DCMAKE_BUILD_TYPE=Release`, with `-s` added to the C and C++ release flags so the binaries are stripped of debug symbols;
- `-DBUILD_GRAPHICAL_EXAMPLES=OFF`;
- `-DBUILD_PYTHON_BINDINGS:bool=true`; Python itself is not installed in the final image, and `PYTHONPATH` is extended with `/usr/local/lib`;
- `-DCMAKE_INSTALL_PREFIX=/opt/librealsense`.

The image also installs the udev rules `99-realsense-libusb.rules` and `99-realsense-d4xx-mipi-dfu.rules` from `config/`. Its default command is `rs-enumerate-devices --compact`, which prints a short summary of the connected devices (`-S` is the short form of `--compact`).

Because `BUILD_EXAMPLES` and `BUILD_TOOLS` default to `ON` (see [[Build Configuration]]), the non-graphical examples and tools are built; see [[Code Samples]] and [[Command Line Tools]].

## Install Docker

Install Docker for your operating system first. The readme links to a tutorial for Ubuntu.

## Use the pre-built image

The readme says the latest image can be pulled from Docker Hub:

```sh
docker pull librealsense/librealsense
```

It adds that the image is available only for x86 at that stage. Check with `docker images` that `librealsense/librealsense` is listed.

## Build the image yourself

Run the script from inside a clone of the repository, so that it can find the latest tag:

```sh
cd scripts/Docker
./build_image.sh
```

[build_image.sh](https://github.com/realsenseai/librealsense/blob/development/scripts/Docker/build_image.sh) takes the latest tag from `git describe --abbrev=0 --tags`, strips the leading `v` to get `LIBRS_VERSION`, and runs `docker build --target librealsense` with that version as a build argument. The image is tagged `librealsense` and `librealsense:<tag>`. The build fails when `LIBRS_VERSION` is empty.

To build for both architectures, use [build_image_multi_platform.sh](https://github.com/realsenseai/librealsense/blob/development/scripts/Docker/build_image_multi_platform.sh). It uses `docker buildx` to build `linux/amd64` and `linux/arm64` images and writes them to the tar files `librealsense-amd64.tar` and `librealsense-arm64.tar`. The script comments warn that the arm64 build is slow.

## Run the container

The container needs access to the host's USB and video devices. Every `docker run` command below mounts `/dev` and adds two device cgroup rules, `c 81:*` and `c 189:*` with `rmw` access. The readme says these rules grant the container access to the USB and UVC resources of the host.

Default command (lists the connected devices):

```sh
docker run -it --rm \
    -v /dev:/dev \
    --device-cgroup-rule "c 81:* rmw" \
    --device-cgroup-rule "c 189:* rmw" \
    librealsense/librealsense
```

A different command, here the SDK's device enumeration tool without the compact option:

```sh
docker run -it --rm \
    -v /dev:/dev \
    --device-cgroup-rule "c 81:* rmw" \
    --device-cgroup-rule "c 189:* rmw" \
    librealsense/librealsense rs-enumerate-devices
```

An interactive shell:

```sh
docker run -it --rm \
    -v /dev:/dev \
    --device-cgroup-rule "c 81:* rmw" \
    --device-cgroup-rule "c 189:* rmw" \
    librealsense/librealsense /bin/bash
```

For an image you built locally, use the image name `librealsense` instead. [run_image.sh](https://github.com/realsenseai/librealsense/blob/development/scripts/Docker/run_image.sh) runs exactly that: the default command with the same device options.

## See also

- [[Installation Linux]]
- [[Build Configuration]]
- [[Python Wrapper]]

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
