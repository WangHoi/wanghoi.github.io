# Initializing kwui

## Library initialization and main loop

Before using kwui, you need to initialize it with `Application::new()`; this connects to the operation system, sets up the locale and performs other initialization tasks. `Application::quit()` exits the application.

Like most GUI toolkits, kwui uses an event-driven programming model. When the application is doing nothing, kwui sits in the “main loop” and waits for input. If the user performs some action - say, a mouse click - then the main loop “wakes up” and delivers an event to kwui. kwui forwards the event to one or more elements in the gui Scene Tree.

User interface elements and input event callbacks are defined in JavaScript modules. After initialize kwui, you `ScriptEngine::load_file(path)` to load and run the root JavaScript entry script.

When elements receive an event, they frequently emit one or more “event callbacks”. Event callbacks calls into JavaScript function callback, that “something interesting happened”.

When your callbacks in JavaScript are invoked, you would typically take some action - for example, when an Open button is clicked you might update application states or perform some tasks with side effects. After a callback finishes, kwui will return to the main loop and await more user input.

Business code is written in Rust to perform network requests, and native operations. `ScriptEngine` is the bridge between Rust and JavaScript.

The main() function for a simple kwui application:

```rust
// src/main.rs
use kwui::{Application, ScriptEngine};

pub fn main() {
    // initialize kwui
    let app = Application::new();
    
    // set resource directory to local folder
    app.set_resource_root_dir(concat!(
        env!("CARGO_MANIFEST_DIR"),
        "/assets"
    ));
    
    // load 'assets/entry.js'
    ScriptEngine::load_file(":/entry.js");

    // run event loop
    app.exec();
}
```
