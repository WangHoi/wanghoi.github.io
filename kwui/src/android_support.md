# Android Support

**kwui** has experimental android support now.

## Requirements

**kwui** is targeting Android 11 (API level 30) or higher, and build for `arm64-v8a` target ABI now.
Android Emulator and ChromeOS are not supported yet.

## Prepare Android development environment

1. Install latest Android Studio
2. Install latest Android SDK and NDK from Android Studio's SDKManager.
3. Set the system environment variable, Example:
```bash
ANDROID_HOME=D:/projects/Android/Sdk
ANDROID_NDK_HOME=D:/projects/Android/Sdk/ndk/27.0.11718014
CARGO_TARGET_AARCH64_LINUX_ANDROID_LINKER=%ANDROID_NDK_HOME%/toolchains/llvm/prebuilt/windows-x86_64/bin/aarch64-linux-android30-clang.cmd
JAVA_HOME=D:/Program Files/Android/Android Studio/jbr
```

## Build android apk

Enter the project directory, which is created by the kwui-cli tool.

```bash
cd test_proj
kwui build apk
```