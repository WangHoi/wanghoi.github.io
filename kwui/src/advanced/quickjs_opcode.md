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
0C      | special_object    | 1: object_type    | `.` => `object`                   | only used at the start of a function, push new specially constructed object
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

TBD...