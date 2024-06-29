# QuickJS VM

QuickJS is a small and embeddable Javascript engine. It supports most of the ES2023 specification including modules, asynchronous generators, proxies and operator overloading.

**kwui** is using QuickJS to define user interface structure, and responds to user input events.

## Features

- Small and easily embeddable stack based VM.
- Fast interpreter with very low startup time.
- Garbage collection using reference counting.
- Small built-in standard library.
- With simplified Rust API.

## Compiling JavaScript

QuickJS compiles JavaScript to VM bytecode, with 3 passes:

1. Parse JavaScript to generate initial bytecode.
2. Resolve scoped variables to local or free variable references.
3. Resolve and back patch jump offsets, peephole optimization.
