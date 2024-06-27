# QuickJS Opcodes

## ByteCode Compatibilities
The bytecode compatibilities of QuickJS VM, depends on macro defines.

We are compiling QuickJS with:

```cpp
// #undef CONFIG_BIGNUM
// #undef CONFIG_DEBUGGER
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
2E      | return_async      |                   | `.` => `.`                        | save VM execution state, return `undefined` to the caller of `JS_CallInternal()` 
2F      | throw             |                   | `except_obj` => `.`               | take off the exception object from stack, set it as VM's current_exception 
30      | throw_error       | 4:atom, 1:type    | `.` => `.`                        | throw an exception with type and error message atom
31      | eval              | 2:argc, 2:scope   | `eval_func, args...` => `func_ret` | take off the `eval` function object and its argments from stack, call function, push function's return value to the stack
32      | apply_eval        | 2:scope           | `array[args], eval_func_obj` => `func_ret` | take off the `eval` function object and its argments from stack, call function, push function's return value to the stack
33      | regex             |                   | `bytecode, pattern` => `regexp_obj` | construct regular expression object from regexp bytecode and match pattern, push to the stack
34      | get_super         |                   | `obj` => `obj_super`              | get object's prototype type object, replace top stack value
35      | import            |                   | `specifier` => `module`           | dynamic module import
36      | check_var         | 4:atom            | `.` => `var_exists`               | check if a global variable exists
37      | get_var_undef     | 4:atom            | `.` => `a`                        | get global variable and push to the stack, push `undefined` if variable not exists
38      | get_var           | 4:atom            | `.` => `a`                        | get global variable and push to the stack, throw if variable not exists
39      | put_var           | 4:atom            | `a` => `.`                        | set global variable from stack stop, normal variable write, if variable not exist, throw 
3A      | put_var_init      | 4:atom            | `a` => `.`                        | set global variable from stack stop, if variable not exist, initialize lexical variable, 
3B      | put_var_strict    | 4:atom            | `a, strict` => `.`                | set global variable from stack stop, if `strict` is true, throw if variable not exists, else normal variable write
3C      | get_ref_value     |                   | `prop, this` => `this[prop]`      | get object's property
3D      | put_ref_value     |                   | `a, prop, this` => `.`            | set object's property, `this[prop] = a`, if `is_strict_mode()`, throw if property not exists
3E      | define_var        | 4:atom, 1:flags   | `.` => ``                         | define global variable
3F      | check_define_var  | 4:atom, 1:flags   | `.` => `.`                        | check global variable re-declaration, flags is 0, DEFINE_GLOBAL_LEX_VAR or DEFINE_GLOBAL_FUNC_VAR
40      | define_func       | 4:atom, 1:flags   | `func_obj` => `.`                 | define global function
41      | get_field         | 4:atom            | `this` => `a`                     | pop `this` object, `push this[prop]` to the stack
42      | get_field2        | 4:atom            | `this` => `a, this`               | push `this[prop]` to the stack
43      | put_field         | 4:atom            | `a, this` => `.`                  | set object property, `this[prop] = a`
44      | get_private_field |                   | `name, obj` => `a`                | get object private property
45      | put_private_field |                   | `name, a, obj` => `.`             | set object private property, `obj[name] = a`
46      | define_private_field |                | `a, name, obj` => `.`             | set object private property, `obj[name] = a`
47      | get_array_el      |                   | `prop, this` => `a`               | get array element, `a = this[prop]`
48      | get_array_el2     |                   | `prop, this` => `a, this`         | get array element, `a = this[prop]`
49      | put_array_el      |                   | `a, prop, this` => `.`            | set array element, `this[prop] = a`
4A      | get_super_value   |                   | `prop, obj, this` => `a`          | get object property, `Reflect.get(obj, prop, this)`, the value of `this` provided for the `obj` if a getter is encountered
4B      | put_super_value   |                   | `a, prop, obj, this` => `.`       | set object property, `Reflect.set(obj, prop, a, this)`, the value of `this` provided for the `obj` if a setter is encountered
4C      | define_field      | 4:atom            | `a, this` => `this`               | define object property, `this[atom] = a`
4D      | set_name          | 4:atom            | `obj` => `obj`                    | set the name of object
4E      | set_name_computed |                   | `obj, name` => `obj, name`        | set the name of object
4F      | set_proto         |                   | `proto, obj` => `obj`      | set the prototype of object
50      | set_home_object   |                   | `func_obj, home_obj` => `func_obj, home_obj` | set the home object of function
51      | define_array_el   |                   | `a, prop, this` => `prop, this`   | define array element, `this[prop] = a`
52      | append            |                   | `enum_obj, pos, array` => `pos, array` | append to array, `array[pos++] = a`
53      | copy_data_properties | 1:mask         | `.` => `.`                        | copy properties value from source to target object, stack offsets encoded in `mask`: 2 bits for target, 3 bits for source, 2 bits for exclusionList
54      | define_method     | 4:atom, 1:flags   | `a, obj` => `obj`                 | define object method, `obj[atom] = a`
55      | define_method_computed | 1:flags      | `a, atom, obj` => `obj`           | define object method with computed method name, `obj[atom] = a`, must come after `define_method`
56      | define_class      | 4:atom, 1:flags   | `class_ctor, parent` => `proto, ctor` | define a class object
57      | define_class_computed | 4:atom, 1:flags | `class_ctor, parent, name` => `proto, ctor` | define a class object with computed class name 
58      | get_loc           | 2:index           | `.` => `a`                        | push local variable `var_buf[index]` to the stack
59      | put_loc           | 2:index           | `a` => `.`                        | set local variable, `var_buf[index] = a`
5A      | set_loc           | 2:index           | `a` => `a`                        | set local variable, `var_buf[index] = a`
5B      | get_arg           | 2:index           | `.` => `a`                        | push function argument `arg_buf[index]` to the stack
5C      | put_arg           | 2:index           | `a` => `.`                        | set function argument, `arg_buf[index] = a`
5D      | set_arg           | 2:index           | `a` => `a`                        | set function argument, `arg_buf[index] = a`
5E      | get_var_ref       | 2:index           | `.` => `a`                        | push variable reference `var_refs[index]` to the stack
5F      | put_var_ref       | 2:index           | `a` => `.`                        | set variable reference, `var_refs[index] = a`
60      | set_var_ref       | 2:index           | `a` => `a`                        | set variable reference, `var_refs[index] = a`
61      | set_loc_uninitialized | 2:index       | `.` => `.`                        | set local variable to `JS_UNINITIALIZED`, `var_buf[index] = uninitialized`
62      | get_loc_check     | 2:index           | `.` => `a`                        | `get_loc` with check for `var_buf[index] != JS_UNINITIALIZED`
63      | put_loc_check     | 2:index           | `a` => `.`                        | `put_loc` with check for `var_buf[index] != JS_UNINITIALIZED`
64      | put_loc_check_init | 2:index          | `a` => `.`                        | `put_loc` with check for `var_buf[index] == JS_UNINITIALIZED`
65      | get_var_ref_check | 2:index           | `.` => `a`                        | `get_var_ref` with check for `var_refs[index] != JS_UNINITIALIZED`
66      | put_var_ref_check | 2:index           | `a` => `.`                        | `put_var_ref` with check for `var_refs[index] != JS_UNINITIALIZED`
67      | put_var_ref_check_init | 2:index      | `a` => `a`                        | `put_var_ref` with check for `var_refs[index] == JS_UNINITIALIZED`
68      | close_loc         | 2:index           | `.` => `.`                        | when leave scope, check to detach current stack frame's variable
69      | if_false          | 4:label           | `cond` => `.`                     | offset current instruction pointer if false, `if (!cond) pc += label`
6A      | if_true           | 4:label           | `cond` => `.`                     | offset current instruction pointer if true, `if (cond) pc += label`
6B      | goto              | 4:label           | `.` => `.`                        | offset current instruction pointer, `pc += label`
6C      | catch             | 4:label           | `.` => `catch_offset`             | push a catch_offset JSValue to stack, `label` is the offset from the start of current function's byte_code_buf
6D      | gosub             | 4:label           | `.` => `catch_offset`             | push a catch_offset JSValue to stack, `label` is the offset from the start of current function's byte_code_buf, and offset current instruction pointer, `pc += label`, used to execute the finally block
6E      | ret               |                   | `label` => `.`                    | set current instruction pointer, `pc = byte_code_buf + label`, used to return from the finally block
6F      | to_object         |                   | `a` => `obj_a`                    | convert top to stack to object
70      | to_propkey        |                   | `a` => `propkey_a`                | convert top to stack to property key, aka int, string, or symbol
71      | to_propkey2       |                   | `a, obj` => `propkey_a, obj`      | test `obj != undefined && obj != null`, convert top to stack to property key, aka int, string, or symbol
72      | with_get_var      | 4:atom, 4:label, 1:is_with | `obj` => `new_obj`       | replace stack top with `obj[atom]`, offset current instruction pointer, `pc += label - 5`
73      | with_put_var      | 4:atom, 4:label, 1:is_with | `obj, a` => `.`          | set object property, `obj[atom] = a`, offset current instruction pointer, `pc += label - 5`
74      | with_delete_var   | 4:atom, 4:label, 1:is_with | `obj` => `success`       | delete object property, and push the delete status to stack, offset current instruction pointer, `pc += label - 5`
75      | with_make_ref     | 4:atom, 4:label, 1:is_with | `obj` => `atom, obj`     | push the property name to stack, offset current instruction pointer, `pc += label - 5`
76      | with_get_ref      | 4:atom, 4:label, 1:is_with | `obj` => `a, obj`        | get object property, `a = obj[atom]`, offset current instruction pointer, `pc += label - 5`
77      | with_get_ref_undef | 4:atom, 4:label, 1:is_with | `obj` => `a, undefined` | get object property, `a = obj[atom]`, set old stack top to `undefined`, offset current instruction pointer, `pc += label - 5`
78      | make_loc_ref      | 4:atom, 2:index   | `.` => `obj`                      | make a new object, and a new local variable reference, set `object[atom] = get_var_ref(index)`
79      | make_arg_ref      | 4:atom, 2:index   | `.` => `obj`                      | make a new object, and a new function argument reference, set `object[atom] = get_var_ref(index)`
7A      | make_var_ref_ref  | 4:atom, 2:index   | `.` => `obj`                      | make a new object, and a free variable reference, set `object[atom] = var_refs[index]`
7B      | make_var_ref      | 4:atom            | `.` => `atom, global_obj`         | construct a reference to a global variable
7C      | for_in_start      |                   | `obj` => `enum_obj`               | construct `for in` iterator from object
7D      | for_of_start      |                   | `obj` => `catch_offset, method, enum_obj` | construct `for of` iterator from object
7E      | for_await_of_start |                  | `obj` => `catch_offset, method, enum_obj` | construct `for await of` iterator from object
7F      | for_in_next       |                   | `enum_obj` => `done, value, enum_obj` | iterator to the next value
80      | for_of_next       | 1:index           | `catch_offset, method, enum_obj` => `done, value, catch_offset, method, enum_obj` | iterator to the next value
81      | iterator_check_object |               | `iter_ret` => `iter_ret`          | check iterator must return an object
82      | iterator_get_value_done |             | `iter_ret` => `done, comp_val`    | check iterator must return an object, fetch the iterator complete value, and iterator done state
83      | iterator_close    |                   | `catch_offset, method, enum_obj` => `.` | finalize iterator and cleanup
84      | iterator_close_return |               | `ret_val, ..., catch_offset, method, enum_obj` => `catch_offset, method, enum_obj, ret_val` | finalize iterator and cleanup
85      | iterator_next     |                   | `a, catch_offset, method, enum_obj` => `b, catch_offset, method, enum_obj` | call iterator next method, `b = iter_obj.method(a)`
86      | iterator_call     | 1:flags           | `a, catch_offset, method, enum_obj` => `ret_flag, b, catch_offset, method, enum_obj` | call iterator next method, `b = iter_obj.method(a)`, set `ret_flag` in accordance with `flags`
87      | initial_yield     |                   | `.` => `.`                        | save VM execution state, return `undefined` to the caller of `JS_CallInternal()` 
88      | yield             |                   | `.` => `.`                        | save VM execution state, return `FUNC_RET_YIELD` to the caller of `JS_CallInternal()` 
89      | yield_star        |                   | `.` => `.`                        | save VM execution state, return `FUNC_RET_YIELD_STAR` to the caller of `JS_CallInternal()` 
8A      | async_yield_star  |                   | `.` => `.`                        | save VM execution state, return `FUNC_RET_YIELD_STAR` to the caller of `JS_CallInternal()` 
8B      | await             |                   | `.` => `.`                        | save VM execution state, return `FUNC_RET_AWAIT` to the caller of `JS_CallInternal()` 
8C      | neg               |                   | `a` => `-a`                       | unary negative operator
8D      | plus              |                   | `a` => `+a`                       | unary plus operator
8E      | dec               |                   | `a` => `a - 1`                    | unary decrement operator
8F      | inc               |                   | `a` => `a + 1`                    | unary increment operator
90      | post_dec          |                   | `a` => `a - 1, a`                 | push `a - 1` to the stack
91      | post_inc          |                   | `a` => `a + 1, a`                 | push `a + 1` to the stack
92      | dec_loc           | 1:index           | `.` => `.`                        | decrement local variable, `var_buf[index] = var_buf[index] - 1`
93      | inc_loc           | 1:index           | `.` => `.`                        | increment local variable, `var_buf[index] = var_buf[index] + 1`
94      | add_loc           | 1:index           | `a` => `.`                        | add `a` to local variable, `var_buf[index] = var_buf[index] + a`
95      | not               |                   | `a` => `~a`                       | binary not operator
96      | lnot              |                   | `a` => `!a`                       | boolean not operator
97      | typeof            |                   | `a` => `typeof a`                 | `typeof` operator
98      | delete            |                   | `prop, obj` => `success`          | delete object property, return whether delete succeeds
99      | delete_var        | 4:atom            | `.` => `success`                  | delete global object's property, return whether delete succeeds
9A      | mul               |                   | `a, b` => `b * a`                 | multiply operator
9B      | div               |                   | `a, b` => `b / a`                 | divide operator
9C      | mod               |                   | `a, b` => `b % a`                 | modulate operator
9D      | add               |                   | `a, b` => `b + a`                 | add operator
9E      | sub               |                   | `a, b` => `b - a`                 | subtraction operator
9F      | pow               |                   | `a, b` => `pow(b, a)`             | power operator
A0      | shl               |                   | `a, b` => `b << a`                | shift logical left operator
A1      | sar               |                   | `a, b` => `b >> a`                | shift arithmetic right operator
A2      | shr               |                   | `a, b` => `b >> a`                | shift logical right operator
A3      | lt                |                   | `a, b` => `b < a`                 | less than compare
A4      | lte               |                   | `a, b` => `b <= a`                | less than or equal compare
A5      | gt                |                   | `a, b` => `b > a`                 | greater compare
A6      | gte               |                   | `a, b` => `b >= a`                | greater or equal compare
A7      | instanceof        |                   | `obj, cls` => `obj instanceof cls` | object is instanceof class
A8      | in                |                   | `obj, prop` => `prop in obj`      | check object has property
A9      | eq                |                   | `a, b` => `b == a`                | equal compare
AA      | neq               |                   | `a, b` => `b != a`                | not equal compare
AB      | strict_eq         |                   | `a, b` => `b === a`               | strict equal compare
AC      | strict_neq        |                   | `a, b` => `b !== a`               | not strict equal compare
AD      | and               |                   | `a, b` => `b & a`                 | binary and operator
AE      | xor               |                   | `a, b` => `b ^ a`                 | binary xor operator
AF      | or                |                   | `a, b` => `b | a`                 | binary or operator
B0      | is_undefined_or_null |                | `a` => `a_is_undefined_or_null`   | check `a === undefined || a === null`
B1      | nop               |                   | `.` => `.`                        | no-operation, VM will skip this opcode
B2      | enter_scope       |                   | `` => ``                        |
B2      | leave_scope               |                   | `` => ``                        |
B2      | label               |                   | `` => ``                        |
B2      | scope_get_var_undef               |                   | `` => ``                        |
B2      | scope_get_var               |                   | `` => ``                        |
B2      | scope_put_var               |                   | `` => ``                        |
B2      | scope_delete_var               |                   | `` => ``                        |
B2      | scope_make_ref               |                   | `` => ``                        |
B2      | scope_get_ref               |                   | `` => ``                        |
B2      | scope_put_var_init               |                   | `` => ``                        |
B2      | scope_get_private_field               |                   | `` => ``                        |
B2      | scope_get_private_field2               |                   | `` => ``                        |
B2      | scope_put_private_field               |                   | `` => ``                        |
B2      | set_class_name               |                   | `` => ``                        |
B2      | line_num               |                   | `` => ``                        |
B2      | push_minus1               |                   | `` => ``                        |
B2      | push_0               |                   | `` => ``                        |
B2      | push_1               |                   | `` => ``                        |
B2      | push_2               |                   | `` => ``                        |
B2      | push_3               |                   | `` => ``                        |
B2      | push_4               |                   | `` => ``                        |
B2      | push_5               |                   | `` => ``                        |
B2      | push_6               |                   | `` => ``                        |
B2      | push_7               |                   | `` => ``                        |
B2      | push_i8               |                   | `` => ``                        |
B2      | push_i16              |                   | `` => ``                        |
B2      | push_const8               |                   | `` => ``                        |
B2      | fclosure8               |                   | `` => ``                        |
B2      | push_empty_string               |                   | `` => ``                        |
B2      | get_loc8               |                   | `` => ``                        |
B2      | put_loc8               |                   | `` => ``                        |
B2      | set_loc8               |                   | `` => ``                        |
B2      | get_loc0               |                   | `` => ``                        |
B2      | get_loc1               |                   | `` => ``                        |
B2      | get_loc2               |                   | `` => ``                        |
B2      | get_loc3               |                   | `` => ``                        |
B2      | put_loc0               |                   | `` => ``                        |
B2      | put_loc1               |                   | `` => ``                        |
B2      | put_loc2               |                   | `` => ``                        |
B2      | put_loc3               |                   | `` => ``                        |
B2      | set_loc0               |                   | `` => ``                        |
B2      | set_loc1               |                   | `` => ``                        |
B2      | set_loc2               |                   | `` => ``                        |
B2      | set_loc3               |                   | `` => ``                        |
B2      | get_arg0               |                   | `` => ``                        |
B2      | get_arg1               |                   | `` => ``                        |
B2      | get_arg2               |                   | `` => ``                        |
B2      | get_arg3               |                   | `` => ``                        |
B2      | put_arg0               |                   | `` => ``                        |
B2      | put_arg1               |                   | `` => ``                        |
B2      | put_arg2               |                   | `` => ``                        |
B2      | put_arg3               |                   | `` => ``                        |
B2      | set_arg0               |                   | `` => ``                        |
B2      | set_arg1               |                   | `` => ``                        |
B2      | set_arg2               |                   | `` => ``                        |
B2      | set_arg3               |                   | `` => ``                        |
B2      | get_var_ref0               |                   | `` => ``                        |
B2      | get_var_ref1               |                   | `` => ``                        |
B2      | get_var_ref2              |                   | `` => ``                        |
B2      | get_var_ref3               |                   | `` => ``                        |
B2      | put_var_ref0               |                   | `` => ``                        |
B2      | put_var_ref1               |                   | `` => ``                        |
B2      | put_var_ref2               |                   | `` => ``                        |
B2      | put_var_ref3               |                   | `` => ``                        |
B2      | set_var_ref0               |                   | `` => ``                        |
B2      | set_var_ref1               |                   | `` => ``                        |
B2      | set_var_ref2               |                   | `` => ``                        |
B2      | set_var_ref3               |                   | `` => ``                        |
B2      | get_length               |                   | `` => ``                        |
B2      | if_false8               |                   | `` => ``                        |
B2      | if_true8               |                   | `` => ``                        |
B2      | goto8               |                   | `` => ``                        |
B2      | goto16               |                   | `` => ``                        |
B2      | call0               |                   | `` => ``                        |
B2      | call1               |                   | `` => ``                        |
B2      | call2               |                   | `` => ``                        |
B2      | call3               |                   | `` => ``                        |
B2      | is_undefined               |                   | `` => ``                        |
B2      | is_null               |                   | `` => ``                        |
B2      | typeof_is_undefined               |                   | `` => ``                        |
B2      | typeof_is_function               |                   | `` => ``                        |


TBD...