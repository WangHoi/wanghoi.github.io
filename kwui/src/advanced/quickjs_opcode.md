# QuickJS Opcodes

## ByteCode Compatibilities
The bytecode compatibilities of QuickJS VM, depends on macro defines.

We are compiling QuickJS with:

```cpp
// #undef CONFIG_BIGNUM
#define CONFIG_DEBUGGER 1
#define SHORT_OPCODES 1
```

* Big number extension disabled.
* Debugger opcodes enabled.
* Extra short opcodes enabled.

## ByteCode References

### Encoding

Each opcode is encoded by 1-byte opcode type, variable length operands. Each operand is little-endian encoded 1, 2, or 4-bytes length integer.

```
+--------+---------------------------+
| opcode | operand_1, operator2, ... |
+--------+---------------------------+
```

Typical operand types:
* index: index into the stack, or some array.
* label: jump label index or offset.
* value: literal value.
* atom: interned string.

### Notation
* `a, b` => `a + b` indicates that the `add` opcode takes two items off the stack (`a` and `b`) and places the sum of these two values on the stack. The leftmost item (`a`) is the top of the stack.
* `.` => `a` indicates that the `push_xxx` opcode takes no stack item, and push a new item to the top of stack.
* All stack descriptions elide subsequent items that may be on the stack. It can be assumed that unspecified stack elements do not influence the semantics of an operation, except when a stack overflow would result.
* The stack size is determined when compiling the script function, and all stack items are `JSValue`.
* Excluding the designated `invalid` opcode, the QuickJS VM currently implements 245 opcodes, 66 of which are short opcodes indicating the number of operands (`push_N`, `get_locN`, `put_locN`, `get_argN`, ...).
* Function arguments on stack are in reverse orders, the first argument is pushed to the stack first, therefore the last argument is on the top of the stack.  

#
| Hex   | Name              | Operands          | Stack      | Notes |
| :---: | :---              | :---              | :---       | :---  |
|       |                   | [bytes]:[type]    | top, bottom|       |
00      | invalid           |                   || never emitted invalid opcode
01      | push_i32          | 4: value          | `.` => `a`                        | push the i32 value
02      | push_const        | 4: index          | `.` => `a`                        | fetch value from the constant pool, push
03      | fclosure          | 4: index          | `.` => `a`                        | must follow push_const, fetch function from the constant pool, bind var refs with current context, push the new closure
04      | push_atom_value   | 4: value          | `.` => `atom_a`                   | push the atom value
05      | private_symbol    | 4: value          | `.` => `symbol_a`                 | construct symbol from atom value, push 
06      | undefined         |                   | `.` => `undefined`                | push `undefined` constant value
07      | null              |                   | `.` => `null`                     | push `null` constant value
08      | push_this         |                   | `.` => `this`                     | only used at the start of a function
09      | push_false        |                   | `.` => `false`                    | push `false` constant value
0A      | push_true         |                   | `.` => `true`                     | push `true` constant value
0B      | object            |                   | `.` => `object_a`                 | push new empty object
0C      | special_object    | 1: obj_type       | `.` => `object`                   | only used at the start of a function, push new specially constructed object
0D      | rest              | 2: value          | `.` => `array[args]`              | push new array from current function's arguments, excluding the first `value` args.
0E      | drop              |                   | `a` => `.`                        | pop the stack
0F      | nip               |                   | `a, b` => `a`                     | remove the second value from the top
10      | nip1              |                   | `a, b, c` => `a, b`               | remove the third value from the top
11      | dup               |                   | `a` => `a, a`                     | duplicate top value
12      | dup1              |                   | `a, b` => `a, b, b`               | duplicate and insert the second value
13      | dup2              |                   | `a, b` => `a, b, a, b`            | duplicate top two values
14      | dup3              |                   | `a, b, c` => `a, b, c, a, b, c`   | duplicate top three values
15      | insert2           |                   | `a, b` => `a, b, a`               | duplicate top value, insert at the third place
16      | insert3           |                   | `a, b, c` => `a, b, c, a`         | duplicate top value, insert at the fourth place
17      | insert4           |                   | `a, b, c, d` => `a, b, c, d, a`   | duplicate top value, insert at the fifth place
18      | perm3             |                   | `a, b, c` => `a, c, b`            | swap the second and third value
19      | perm4             |                   | `a, b, c, d` => `a, d, c, b`      | swap the second and fourth value
1A      | perm5             |                   | `a, b, c, d, e` => `a, e, c, d, b`| swap the second and fifth value
1B      | swap              |                   | `a, b` => `b, a`                  | swap the first and second value
1C      | swap2             |                   | `a, b, c, d` => `c, d, a, b`      | swap the first and third value, the second and fourth value
1D      | rot3l             |                   | `a, b, c` => `c, a, b`            | take off the third value, and push it to the top
1E      | rot3r             |                   | `a, b, c` => `b, c, a`            | take off the first value, and insert it to the third place
1F      | rot4l             |                   | `a, b, c, d` => `d, a, b, c`      | take off the fourth value, and push it to the top
20      | rot5l             |                   | `a, b, c, d, e` => `e, a, b, c, d`| take off the fifth value, and push it to the top
21      | call_constructor  | 2:argc            | `new_target, func_obj, args...` => `func_ret`| take off the constructor target value, function object and its argments from stack, call object constructor, push constructor's return value to the stack
22      | call              | 2:argc                 | `func_obj, args...` => `func_ret` | take off the function object and its argments from stack, call function, push function's return value to the stack
23      | tail_call         | 2:argc            | `func_obj, args...` => `.`        | take off the function object and its argments from stack, call function, return the function's return value to the caller of `JS_CallInternal()`
24      | call_method       | 2:argc            | `func_obj, this, args...` => `func_ret` | take off the function object, `this` object, and its argments from stack, call method, push method's return value to the stack
25      | tail_call_method  | 2:argc            | `func_obj, this, args...` => `.`  | take off the function object, `this` object, and its argments from stack, call method, return the method's return value to the caller of `JS_CallInternal()`
26      | array_from        | 2:argc            | `args...` => `[args]`             | take off the function arguments from stack, push new array of arguments to the stack
27      | apply             | 2:magic           | `array[args], this, func_obj` => `func_ret` | call function or method with array of arguments, magic value: 0 = normal apply, 1 = apply for constructor, 2 = Reflect.apply, push return value of function to the stack
28      | return            |                   | `func_ret` => `.`                 | take off the function return value from stack, and return to the caller of `JS_CallInternal()` 
29      | return_undef      |                   | `.` => `.`                        | return `undefined` to the caller of `JS_CallInternal()` 
2A      | check_ctor_return |                   | `a` => `is_ctor_return, a`        | check a is `undefined` or `object`, derived class constructor must return an object or undefined 
2B      | check_ctor        |                   | `.` => `.`                        | check calling function's argument, `JSCallInternal(new_target)`, check `new_target` != `undefined`, class constructors must be invoked with 'new' 
2C      | check_brand       |                   | `func, obj` => `func, obj`        | check home object of `func` and `obj` has same brand, if failed, exception returned to the caller to `JS_CallInternal()`
2D      | add_brand         |                   | `home_obj, obj` => `.`            | add a brand property to `object`, which points to the brand of `home_obj`
2E      | return_async      |                   | `.` => `.`                        | save VM stack execution state, return `undefined` to the caller of `JS_CallInternal()` 
2F      | throw             |                   | `except_obj` => `.`               | take off the exception object from stack, set it as VM's current_exception 
30      | throw_error       | 4:atom, 1:type    | `.` => `.`                        | throw an exception with type and error message atom

TBD...