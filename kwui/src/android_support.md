# Android Support

**kwui** has experimental android support now.

## Prepare Android development environment

1. Install latest Android Studio
2. Install latest Android SDK and NDK from Android Studio's SDKManager.
3. Set the system environment variable, Example:
```bash
ANDROID_HOME=D:/projects/Android/Sdk
ANDROID_NDK_HOME=D:/projects/Android/Sdk/ndk/27.0.11718014
CARGO_TARGET_AARCH64_LINUX_ANDROID_LINKER=%ANDROID_NDK_HOME%/toolchains/llvm/prebuilt/windows-x86_64/bin/aarch64-linux-android30-clang.cmd
```

## Build android apk

Enter the project directory, which is created by the kwui-cli tool.

```bash
cd test_proj
kwui build apk
```