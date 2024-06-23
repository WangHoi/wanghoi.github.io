# Quick Start

**kwui** supports `Windows` host only, targeting `Windows` and `Android`.
>NOTE: `Linux` and `OpenHarmony(OHOS)` support in the future.

## Installation

1. Install the Rust development environments.
2. Install **kwui-cli** tool.
    ```rust
    cargo install kwui-cli
    ```

## First project

Let's create a hello world project `my_proj`:

1. Create project from template.
    ```bash
    # Creates a new Rust project 'my_proj' under current directory
    kwui new my_proj
    cd my_proj
    ```
2. Build and run.
    ```
    cargo run
    ```
> **_NOTE_**
>
> **kwui** is using prebuilt library to accelerate the build process. 
> 
> If you are in a country where access to GitHub is difficult or slow,
> set the following environment variable:
> - KWUI_BINARIES_URL=https://gitee.com/wanghoi/kwui-binaries/releases/download/{tag}/kwui-binaries-{key}.tar.gz
> - KWUI_TEMPLATES_URL=https://gitee.com/wanghoi/kwui-binaries/releases/download/{tag}/kwui-templates-{key}.tar.gz

## Porting to Android

1. Prepare the Android development environment.
2. Build the project for Android.
    ```
    # Build 'app-debug.apk'
    kwui build apk
    ```
3. Install the resulting APK to your phone.
    > NOTE: only supports `arm64-v8a` abi now.