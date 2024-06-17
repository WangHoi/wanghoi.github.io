# Introduction

[kwui](https://github.com/wanghoi/kwui-rs) is a gui toolkit for creating graphical user interfaces. It works on Windows, and Android. **kwui** is released under the terms of the GNU Lesser General Public License, which allows for flexible licensing of client applications. **kwui** is inspired by [rust-sciter](https://github.com/sciter-sdk/rust-sciter), that use JavaScript and CSS to describe the UI, and Rust for business logic.

The kwui toolkit contains “widgets”: GUI components such as buttons, text input, or windows.

**kwui** depends the following libraries:

- **Skia**: Skia is a 2D graphics library with support for multiple output devices. More information available on the [Skia website](https://skia.org/).
- **Direct2D**: Direct2D is a 2D graphics library for windows platform.
- **QuickJS**: QuickJS is a small and embeddable JavaScript engine.

**kwui** is divided into three parts:

- **kwui-rs**: kwui-rs is the Rust binding to the C++ **kwui-sys** library.
    - Provides simple and type-safe API.
    - Integrate with cargo crates ecosystem.

- **kwui-sys**: kwui-sys is written in C++. It inter-operates with underline operating system, render the user interface, and handle user input events. It's pretty complex, some features:
    - Renderer: `Direct2D` or `Skia`
    - Layout and style: `CSS`
    - Scripting: builtin `JavaScript` engine with `JSX` extension support.
    - Data model: `React Hooks` alike JavaScript functional components.
    - Platform-dependent resource system

- **kwui-cli**: kwui-cli is a command line tool.
    - Create project template.
    - Package application resources.
    - Provide build assistance for Windows and Android target platform.
