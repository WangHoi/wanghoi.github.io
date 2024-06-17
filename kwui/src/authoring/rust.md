# Rust and JavaScript Interop

## Basic sample

```rust
use kwui::{Application, ScriptEngine, ScriptValue};

pub fn main() {
    let app = Application::new();
    
    // set resource directory to local folder
    app.set_resource_root_dir(concat!(
        env!("CARGO_MANIFEST_DIR"),
        "/assets"
    ));
    
    // load 'assets/entry.js'
    ScriptEngine::load_file(":/entry.js");

    // call JavaScript `function add(a, b) -> sum` from Rust
    let sum = ScriptEngine::call_global_function("script_add", &[
        ScriptValue::new_int(1),
        ScriptValue::new_int(2),
    ]).to_int();

    // export Rust function to JavaScript
    ScriptEngine::add_global_function("rust_add", rust_add);

    // run event loop
    app.exec();
}

fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}
```

```javascript
// call Rust function from JavaScript
console.log("sum:", rust_add(1, 2));
```

## References

[Rust API documentation](https://docs.rs/kwui)
