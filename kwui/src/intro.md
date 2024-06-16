# Introduction

**kwui** is a gui toolkit for Rust developer.

It is created to make developer's life easier,
writing simple GUI programs efficiently,
and support both desktop and mobile platforms.

## Architecture

**kwui** is made up by three components.

- **kwui-rs** is the Rust binding to the C++ **kwui-sys** library.
    - Provides simple and type-safe API.
    - Integrate with cargo crates ecosystem.

- **kwui-sys** is written in C++. It inter-operates with underline operating system, render the user interface, and handle user input events. It's pretty complex, some features:
    - Renderer: `Direct2D` or `Skia`
    - Layout and style: `CSS`
    - Scripting: builtin `JavaScript` engine with `JSX` extension support.
    - Data model: `React Hooks` alike JavaScript functional components.
    - Platform-dependent resource system

- **kwui-cli** is a command line tool.
    - Create project template.
    - Package application resources.
    - Provide build assistance for Windows and Android target platform.

### How it works (basically)

1. Rust: create and initialize **kwui** application and scripting environment.<br /> 
  Register Rust-JS script bindings, finally run the JavaScript entry js.
2. JavaScript: Script side initialization, finally provide the root component.
3. **kwui** takes care of the rendering and handle the UI events.

### Developer workflow

**kwui** is designed with typical developer workflow in mind:
1. Create project template with **kwui-cli** tool.
2. Develop GUI-less code first.
3. Add **kwui** dependency, develop and test GUI glue code on the Windows platform.
4. Build and test on other(Android, etc.) platforms.
