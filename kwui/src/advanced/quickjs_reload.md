# Live Reload

**kwui** has a built-in live reload system. It allows you to change your JavaScript user interface source code and view the result in realtime.

> Live Reload is currently only available for Windows Debug builds.

## Usage

1. Load resource from local directory
    ```rust
    let app = Application::new();
    if cfg!(all(target_os = "windows", debug_assertions)) {
        app.set_resource_root_dir("{your assets dir}");
    }
    ```
    Then asset URLs like `:/entry.js` will be mapped to `{your assets dir}/entry.js`.

2. While the application is running, pressing `F5` to trigger Live Reload of current Dialog, the user interface will be re-built.
    > Rust-side states are preserved, JavaScript module-level and component-level variables and states are re-initialized during reload.

## How it works

- Increase application's script version. 
- Reload entry module script
    - `import` will also import a new version of dependent JavaScript modules.
- Call `builder()` to build a new root component tree.
    - Child components in other JavaScript modules are built recursively.
- Patch old user interface with new root component tree.
- Garbage Collection will recycle old JavaScript module states later.