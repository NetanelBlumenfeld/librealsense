This page explains how to build the RealSense SDK for Android and how to use it from an Android application. The Android wrapper packages the SDK as an Android Archive (AAR) that lets you write both native (C/C++) and Java applications. It is for Android developers who want to stream from a RealSense camera attached through USB OTG. The sources are in [wrappers/android](https://github.com/realsenseai/librealsense/tree/development/wrappers/android) and the overview is in [doc/android.md](https://github.com/realsenseai/librealsense/blob/development/doc/android.md).

## What you get

- `librealsense.aar`: the native SDK built with the RSUSB backend, plus a JNI based Java API (classes such as `Pipeline`, `Config`, `Device`, `FrameSet` and the processing filters, in package `com.intel.realsense.librealsense`).
- The AAR also takes care of asking for camera access permission, according to the repository guide.
- Example applications and a camera tool app (see below).

The repository guide states that RealSense has not explicitly validated the SDK on Android, although it is expected to work, and that the calibration and firmware update tools used in production are not available on Android. Android builds turn off the unit tests, the C++ examples, the tools and the graphical examples (`CMake/android_config.cmake`).

## Requirements

- An Android device that supports USB OTG, and a USB 3 OTG cable.
- [Android Studio](https://developer.android.com/studio), or only a JDK and the Gradle wrapper for command-line builds.
- The versions the project is configured with (`wrappers/android/build.gradle` and `librealsense/build.gradle`): minimum SDK 24, target and compile SDK 35, Android Gradle plugin 8.9.1, NDK 28.0.13004108.

## Build the AAR

### With Gradle

From `wrappers/android` in the repository:

```sh
./gradlew assembleRelease
```

On a Windows host run `gradlew assembleRelease`. The AAR is written to `wrappers/android/librealsense/build/outputs/aar`.

### With Android Studio

1. Open Android Studio and choose File, Open, and select [build.gradle](https://github.com/realsenseai/librealsense/blob/development/wrappers/android/build.gradle) in `wrappers/android`.
2. In the Gradle tasks run `:librealsense`, `build`, `assembleRelease`.
3. Take the AAR from the same output folder as above.

The library is built for the ABIs `arm64-v8a` and `x86_64`, with `-DFORCE_RSUSB_BACKEND=TRUE` (`wrappers/android/librealsense/build.gradle`). The repository readme still says four ABIs; the Gradle file lists two.

You can add the generated AAR to your own project as described in the [Android documentation](https://developer.android.com/studio/projects/android-library#AddDependency).

## Use the SDK in an application

Two stand-alone examples show only the Gradle configuration and minimal code: the [Java example](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/examples/java_example) and the [native example](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/examples/native_example). Both declare the library as `com.intel.realsense:librealsense` in their module `build.gradle` (the Java example uses `2.+@aar`; pin a specific version in your own project).

The Java example demonstrates basic streaming with the `Pipeline` class and reacting to device connection events. Its manifest requests the camera permission:

```xml
<uses-permission android:name="android.permission.CAMERA"/>
```

The repository readme says this permission is needed by Android 9 and later for apps that target those versions.

## Examples and tools in the Android project

`wrappers/android/settings.gradle` includes these modules:

| Module | Folder |
|---|---|
| Library | [`librealsense`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/librealsense) |
| Camera tool app | [`tools/camera`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/tools/camera) |
| Capture | [`examples/capture`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/examples/capture) |
| Playback | [`examples/playback`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/examples/playback) |
| Recording | [`examples/recording`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/examples/recording) |
| Processing | [`examples/processing`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/examples/processing) |
| Multicam | [`examples/multicam`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/examples/multicam) |
| Sensor | [`examples/sensor`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/examples/sensor) |
| Stream pipeline callback | [`examples/stream_pipeline_callback`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/examples/stream_pipeline_callback) |

To run one, select it in the Android Studio configuration box and start it while the phone is connected. The folder [`code_fragments`](https://github.com/realsenseai/librealsense/tree/development/wrappers/android/code_fragments) holds snippets that show API usage; its readme warns they are not meant to compile.

## See also

- [[Build Configuration]], for the `FORCE_RSUSB_BACKEND` option
- [[Record and Playback]]
- [[Installation Linux]]

<!-- wikigen:footer -->
_Maintained from librealsense `development`; last checked at commit [`5c14a0b`](https://github.com/realsenseai/librealsense/commit/5c14a0b5309a878032e7196889df475adf8c4f7d)._
<!-- /wikigen:footer -->
