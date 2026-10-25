This page explains how to install the RealSense SDK on Windows 10 and 11: using the pre-built binaries from GitHub Releases, building from source with CMake and Visual Studio, and enabling per-frame metadata. It also notes the legacy Windows 7 path. The repository guides it is based on are [distribution_windows.md](https://github.com/realsenseai/librealsense/blob/development/doc/distribution_windows.md), [installation_windows.md](https://github.com/realsenseai/librealsense/blob/development/doc/installation_windows.md) and [installation_win7.md](https://github.com/realsenseai/librealsense/blob/development/doc/installation_win7.md).

The SDK team does not recommend or support installation in a virtual machine, because of the USB 3.0 translation layer between the hardware and the VM.

## Use the pre-built binaries

1. Plug the camera into a USB 3 port. It should appear in the Device Manager.
2. Open the [latest stable release](https://github.com/realsenseai/librealsense/releases/latest) on GitHub and go to its **Assets** section.
3. Download and run the RealSense Viewer to explore the depth data ([[Realsense Viewer Manual]]), or download and run the SDK installer to get the Viewer together with development packages and the samples solution.
4. After installing the SDK, open the *RealSense Samples* solution and press `F5` to compile and run the demos.

## Build from source

The guide requires [CMake](https://cmake.org/download/) (v3.10 or newer) and Visual Studio 2019 or newer; MSVC 2015 and older are not fully compatible with the C++14 feature set.

1. Configure the project with CMake (`cmake-gui` or the command line). Select `BUILD_EXAMPLES` if you want the samples.
2. Generate the Visual Studio solution and build it.

Other options are listed on [[Build Configuration]].

## Enable frame metadata

Metadata (per-frame attributes such as timestamps and exposure) needs a registry entry for every distinct video device on Windows. See [[Frame Metadata]] for what metadata is.

Prerequisites per the guide: Windows 10 with administrator login, and Windows SDK 10 (10.0.15063) or later. Run `winver` to check the OS: Windows 10 build 15063 or later (17763+ recommended) or Windows 11.

### With the script

[`scripts/realsense_metadata_win10.ps1`](https://github.com/realsenseai/librealsense/blob/development/scripts/realsense_metadata_win10.ps1) writes the registry values for you. Start PowerShell **as administrator**, go to the script's folder and run:

```
PS > .\realsense_metadata_win10.ps1 -op install
```

| `-op` value | Effect |
|---|---|
| `install` | Add the registry keys for the connected RealSense devices (the default) |
| `install_all` | Add keys for all RealSense devices previously connected |
| `remove` | Remove the keys for the connected RealSense devices |
| `remove_all` | Remove the keys for all RealSense devices previously connected |

- If PowerShell reports that running scripts is disabled, run `Set-ExecutionPolicy RemoteSigned`, answer `Y` and run the script again.
- The keys are unique per device, so run the script each time a new RealSense device is attached to the PC.

### Manually

If the script cannot run, set the values by hand. For every interface of the device (find them in the Device Manager, e.g. *RealSense Camera D415 Depth* and *RealSense Camera D415 RGB*), open the entry whose name matches the interface's device instance path under both of these registry branches:

- `HKLM\SYSTEM\CurrentControlSet\Control\DeviceClasses\{e5323777-f976-4f5b-9b55-b94699c46e44}`
- `HKLM\SYSTEM\CurrentControlSet\Control\DeviceClasses\{65E8773D-8F56-11D0-A3B9-00A0C9223196}`

and under `#GLOBAL` > `Device Parameters` add a 32-bit DWORD `MetadataBufferSizeInKB0` with the value 5. For the zero interface (`MI_00`) of the multi-pin devices listed in the script (RS400 devices among them), also add a 32-bit DWORD `MetadataBufferSizeInKB1` with the value 5.

### Compile with metadata support

The build detects the Windows SDK version and enables or disables UVC metadata accordingly. If the installed SDK cannot provide metadata, the build prints a notification that the library is compiled without it. Set the CMake option `ENFORCE_METADATA` (default `OFF`, Windows only) to make the build stop instead. If several Windows SDK versions are installed, CMake selects the latest; if that fails, retarget the solution to the right SDK version in Visual Studio.

## Windows 7 (legacy)

Windows 7 is no longer maintained and not all features are enabled on it. The guide installs a dedicated Windows 7 driver with the old SDK installer, lists the D410, D415, D430 and D435 as supported there, and notes that multiple cameras are not validated. Builds for Windows 7 differ from Windows 10 builds; to build for Windows 7 from Windows 10 the guide passes `-DFORCE_WINUSB_UVC=true`, an option that the build now lists as deprecated in favour of `FORCE_RSUSB_BACKEND`. See [installation_win7.md](https://github.com/realsenseai/librealsense/blob/development/doc/installation_win7.md) for the steps.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| The camera is not recognized by the SDK | The SDK has no access to the camera | In Windows, turn on *Allow desktop apps to access your camera* (Settings > Privacy > Camera) |

See also [[Troubleshooting Q&A]].

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
