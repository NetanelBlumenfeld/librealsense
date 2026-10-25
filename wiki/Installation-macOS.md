This page explains how to build the RealSense SDK from source on macOS and how to bundle the library with your own application. It is for anyone who wants to run the SDK examples and tools on a Mac. On macOS the SDK always uses the user-space RSUSB backend; the repository guide is [installation_osx.md](https://github.com/realsenseai/librealsense/blob/development/doc/installation_osx.md).

## Before you start

- macOS support for the full range of SDK functionality is not complete. The repository guide points users of the legacy R200 and ZR300 cameras to the [legacy branch](https://github.com/realsenseai/librealsense/tree/legacy).
- The build system turns `FORCE_RSUSB_BACKEND` on by itself when it detects macOS (`CMake/unix_config.cmake`), so the RSUSB (libusb based) backend is the only one available. Passing `-DFORCE_RSUSB_BACKEND=ON` explicitly, as the steps below do, is harmless.
- CMake 3.10 is the minimum version the top-level `CMakeLists.txt` asks for.

## Build from source

1. Install the command-line developer tools with `sudo xcode-select --install`, or install Xcode from the App Store.
2. Install the [Homebrew](https://brew.sh/) package manager.
3. Install the build dependencies:
   ```sh
   brew install cmake libusb pkg-config openssl
   ```
4. Clone the repository:
   ```sh
   git clone https://github.com/realsenseai/librealsense.git
   cd librealsense
   ```
5. Configure the build:
   ```sh
   mkdir build && cd build
   sudo xcode-select --reset
   cmake .. -DBUILD_EXAMPLES=true -DBUILD_GRAPHICAL_EXAMPLES=true -DFORCE_RSUSB_BACKEND=ON
   ```
6. Build:
   ```sh
   make -j2
   ```

All CMake options are listed on [[Build Configuration]].

### Options that matter on macOS

| Option | Default on macOS | Note |
|---|---|---|
| `BUILD_EXAMPLES` | `ON` | Builds the examples that are not graphical. |
| `BUILD_GRAPHICAL_EXAMPLES` | `ON` | Builds the graphical examples (Viewer and Depth Quality Tool). |
| `FORCE_RSUSB_BACKEND` | `OFF`, then forced `ON` on macOS | Described in `CMake/lrs_options.cmake` as "mandatory for Win7/MacOS/Android". |
| `CHECK_FOR_UPDATES` | `OFF` | The option's comment says the feature needs OpenSSL, which macOS normally does not provide. |
| `ENABLE_AI_ASSISTANT` | `OFF` | Same per-OS default as `CHECK_FOR_UPDATES`. |

When the build links on macOS it needs the CoreFoundation, IOKit and Security frameworks and stops with an error if one of them is missing (`CMake/external_libusb.cmake`).

### Known build problems

- `ld: library not found for -lusb-1.0`, in the terminal or in Xcode. The repository guide works around it by setting the library path for the session: `/bin/launchctl setenv LIBRARY_PATH /usr/local/lib`.
- `Could NOT find OpenSSL`, usually after setting `-DCHECK_FOR_UPDATES=ON`. The repository guide works around it with `` export OPENSSL_ROOT_DIR=`brew --prefix openssl` ``. The SDK's macOS CI job sets `OPENSSL_ROOT_DIR` for the same reason.

## Run the examples

After the build, run the programs from the build folder. The executables are named as below; on Unix-like systems CMake places them in a folder named after the build type inside the build folder (`CMake/unix_config.cmake`).

```sh
rs-hello-realsense
rs-enumerate-devices
rs-multicam
rs-capture
```

See [[Code Samples]] for the list of examples and [[Command Line Tools]] for the tools.

## Package your application

The SDK needs libusb at run time, so an application has to ship it. The repository guide gives these steps:

1. Point the library at a relative libusb location with `install_name_tool`:
   ```sh
   install_name_tool -change /usr/local/opt/libusb/lib/libusb-1.0.0.dylib @rpath/libusb-1.0.0.dylib librealsense2.dylib
   ```
2. Copy `libusb-1.0.0.dylib` and `librealsense2.dylib` into your application's `Frameworks` folder.

## See also

- [[Build Configuration]]
- [[Installation Linux]] and [[Installation Windows]]

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
