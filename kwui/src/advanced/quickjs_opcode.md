# QuickJS Opcodes

## ByteCode Compatibilities

The bytecode compatibilities of QuickJS VM, depends on macro defines.

We are compiling QuickJS with:

```cpp
// #undef CONFIG_ATOMICS
// #undef CONFIG_BIGNUM
// #undef CONFIG_DEBUGGER
#define SHORT_OPCODES 1
```

* Atomics support disabled.
* Big number extension disabled.
* Debugger opcodes disabled.
* Extra short opcodes enabled.

## ByteCode Module Format

After the JavaScript module file is compiled, the module's bytecode can be serialized, and loaded later.

Lexical-scope child functions are serialized recursively as parent function's constants.

## Notation

* multi-bytes values are serialized in little endian order.
* `ATOM` - interned string - when serialization, converted to zero started index.
* `xB` - length is `x` bytes
* `LEB128` - Little Endian Base 128 – length is a variable-length encoding format designed to store arbitrarily large
  integers in a small number of bytes. There are two variations: unsigned LEB128 and signed LEB128. These vary slightly,
  so a user program that wants to decode LEB128 values would explictly choose the appropriate unsigned or signed
  methods.
* `STRING` - string length, followed by utf-8 or utf1-16 string data

### Compiled Module Specification

* The compiled script module contains:
    ```
    1B: version
    LEB128: number of atoms
        [STRING][STRING][STRING] : an array of atom strings
    MODULE: script module
        FUNCTION - module root function
    ```
* STRING
    ```
    LEB128: for utf-8 string, `length << 1`, for utf-16 string, `(length << 2) + 1`
    [1B] or [2B]: utf-8 or utf-16 string data
    ```
* ATOM
    ```
    LEB128: for intger atom, `(atom << 1) | 1`, for string atom, converted to `(first_atom + index) << 1`
    ```
* MODULE
    ```
    1B: module tag, hex `0x0f`
    ATOM: module name
    LEB128: number of requested module entries
        [ATOM]: requested module name
    LEB128: number of exported module entries
        [LOCAL_EXPORT] or [INDIRECT_EXPORT]: local variable/function export or indirect export
    LEB128: number of star exported module entries
        [LEB128]: star exported module indexes
    LEB128: number of imported module entries
        [IMPORT]: imported modules
    ```
* FUNCTION
    ```
    1B: function tag, hex `0x0e`
    2B: flag, from lsb to msb:
        1b: has_prototype
        1b: has_simple_paramter_list
        1b: is_derived_class_constructor
        1b: need_home_object
        2b: func_kind
        1b: new_target_allowed
        1b: super_call_allowed
        1b: super_allowed
        1b: arguments_allowed
        1b: has_debug
        1b: backtrace_barrier
        4b: _unused_
    1B: js mode, 1: strict
    ATOM: function name
    LEB128: argument count
    LEB128: variable count
    LEB128: defined argument count
    LEB128: stack size
    LEB128: closure variable count
    LEB128: constant `JS_VALUE` pool count
    LEB128: bytecode length
    LEB128: `(arg_count + var_count)`
    [VAR_DEF]: `(arg_count + var_count)` variable definitions
    [CLOSURE_VAR]: `closure_var_count` closure variable definitions
    FUNCTION_BYTECODES - raw function bytecodes, atoms inside operands and converted to atom indices
    DEBUG_INFO - optional script debug information
    [JS_VALUE] - `cpool_count` - script constant values
    ```
* DEBUG_INFO
    ```
    ATOM - script filename
    LEB128 - script start line number
    LEB128 - `pc2line_len` - size of the opcodes to line number table
        [1B] - `pc2line_buf`
    ```

* JS_VALUE
    ```
    FUNCTION, STRING, OBJECT, or ARRAY
    ```

## ByteCode References

### Encoding

Each opcode is encoded by 1-byte opcode type, variable length operands. Each operand is little-endian encoded 1, 2, or
4-bytes length integer.

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

* `a, b` => `a + b` indicates that the `add` opcode takes two items off the stack (`a` and `b`) and places the sum of
  these two values on the stack. The leftmost item (`a`) is the top of the stack.
* `.` => `a` indicates that the `push_xxx` opcode takes no stack item, and push a new item to the top of stack.
* All stack descriptions elide subsequent items that may be on the stack. It can be assumed that unspecified stack
  elements do not influence the semantics of an operation, except when a stack overflow would result.
* The stack size is determined when compiling the script function, and all stack items are `JSValue`.
* Excluding the designated `invalid` opcode, the QuickJS VM currently implements 245 opcodes, 66 of which are short
  opcodes indicating the number of operands (`push_N`, `get_locN`, `put_locN`, `get_argN`, ...).
* Function arguments on stack are in reverse orders, the first argument is pushed to the stack first, therefore the last
  argument is on the top of the stack.

<table>
<thead>
        <tr>
            <th style="text-align: center">Hex</th>
            <th style="text-align: left">Name</th>
            <th style="text-align: left">Operands</th>
            <th style="text-align: left">Stack</th>
        </tr>
</thead>
<tbody>
        <tr>
            <td style="text-align: center"></td>
            <td style="text-align: left">[notes]</td>
            <td style="text-align: left">[bytes]:[type]</td>
            <td style="text-align: left">top, bottom</td>
        </tr>
<tr>
<td>00</td>
<td><strong>invalid</strong></td>
<td></td>
<td></td>
</tr>
<tr>
<td></td>
<td colspan="3">never emitted invalid opcode</td>
</tr>
<tr>
<td>01</td>
<td><strong>push_i32</strong></td>
<td>4: value</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push the i32 value</td>
</tr>
<tr>
<td>02</td>
<td><strong>push_const</strong></td>
<td>4: index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">fetch value from the constant pool, push</td>
</tr>
<tr>
<td>03</td>
<td><strong>fclosure</strong></td>
<td>4: index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">must follow push_const, fetch function from the constant
                pool, bind var refs with current context, push the new closure</td>
</tr>
<tr>
<td>04</td>
<td><strong>push_atom_value</strong></td>
<td>4: value</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">atom_a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push the atom value</td>
</tr>
<tr>
<td>05</td>
<td><strong>private_symbol</strong></td>
<td>4: value</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">
                symbol_a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">construct symbol from atom value, push</td>
</tr>
<tr>
<td>06</td>
<td><strong>undefined</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">
                undefined</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push <code class="hljs">undefined</code> constant value</td>
</tr>
<tr>
<td>07</td>
<td><strong>null</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">null</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push <code class="hljs">null</code> constant value</td>
</tr>
<tr>
<td>08</td>
<td><strong>push_this</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">this</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">only used at the start of a function</td>
</tr>
<tr>
<td>09</td>
<td><strong>push_false</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">false</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push <code class="hljs">false</code> constant value</td>
</tr>
<tr>
<td>0A</td>
<td><strong>push_true</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">true</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push <code class="hljs">true</code> constant value</td>
</tr>
<tr>
<td>0B</td>
<td><strong>object</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">
                object_a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push new empty object</td>
</tr>
<tr>
<td>0C</td>
<td><strong>special_object</strong></td>
<td>1: obj_type</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">object</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">only used at the start of a function, push new specially
                constructed object</td>
</tr>
<tr>
<td>0D</td>
<td><strong>rest</strong></td>
<td>2: value</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">
                array[args]</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push new array from current function's arguments, excluding
                the first <code class="hljs">value</code> args.</td>
</tr>
<tr>
<td>0E</td>
<td><strong>drop</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">pop the stack</td>
</tr>
<tr>
<td>0F</td>
<td><strong>nip</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">remove the second value from the top</td>
</tr>
<tr>
<td>10</td>
<td><strong>nip1</strong></td>
<td></td>
<td><code class="hljs">a, b, c</code> =&gt; <code class="hljs">a,
                b</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">remove the third value from the top</td>
</tr>
<tr>
<td>11</td>
<td><strong>dup</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a, a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">duplicate top value</td>
</tr>
<tr>
<td>12</td>
<td><strong>dup1</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">a,
                b, b</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">duplicate and insert the second value</td>
</tr>
<tr>
<td>13</td>
<td><strong>dup2</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">a,
                b, a, b</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">duplicate top two values</td>
</tr>
<tr>
<td>14</td>
<td><strong>dup3</strong></td>
<td></td>
<td><code class="hljs">a, b, c</code> =&gt; <code class="hljs">a,
                b, c, a, b, c</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">duplicate top three values</td>
</tr>
<tr>
<td>15</td>
<td><strong>insert2</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">a,
                b, a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">duplicate top value, insert at the third place</td>
</tr>
<tr>
<td>16</td>
<td><strong>insert3</strong></td>
<td></td>
<td><code class="hljs">a, b, c</code> =&gt; <code class="hljs">a,
                b, c, a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">duplicate top value, insert at the fourth place</td>
</tr>
<tr>
<td>17</td>
<td><strong>insert4</strong></td>
<td></td>
<td><code class="hljs">a, b, c, d</code> =&gt; <code class="hljs">a, b, c, d, a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">duplicate top value, insert at the fifth place</td>
</tr>
<tr>
<td>18</td>
<td><strong>perm3</strong></td>
<td></td>
<td><code class="hljs">a, b, c</code> =&gt; <code class="hljs">a,
                c, b</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">swap the second and third value</td>
</tr>
<tr>
<td>19</td>
<td><strong>perm4</strong></td>
<td></td>
<td><code class="hljs">a, b, c, d</code> =&gt; <code class="hljs">a, d, c, b</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">swap the second and fourth value</td>
</tr>
<tr>
<td>1A</td>
<td><strong>perm5</strong></td>
<td></td>
<td><code class="hljs">a, b, c, d, e</code> =&gt; <code class="hljs">a, e, c, d, b</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">swap the second and fifth value</td>
</tr>
<tr>
<td>1B</td>
<td><strong>swap</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b,
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">swap the first and second value</td>
</tr>
<tr>
<td>1C</td>
<td><strong>swap2</strong></td>
<td></td>
<td><code class="hljs">a, b, c, d</code> =&gt; <code class="hljs">c, d, a, b</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">swap the first and third value, the second and fourth value</td>
</tr>
<tr>
<td>1D</td>
<td><strong>rot3l</strong></td>
<td></td>
<td><code class="hljs">a, b, c</code> =&gt; <code class="hljs">c,
                a, b</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the third value, and push it to the top</td>
</tr>
<tr>
<td>1E</td>
<td><strong>rot3r</strong></td>
<td></td>
<td><code class="hljs">a, b, c</code> =&gt; <code class="hljs">b,
                c, a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the first value, and insert it to the third place</td>
</tr>
<tr>
<td>1F</td>
<td><strong>rot4l</strong></td>
<td></td>
<td><code class="hljs">a, b, c, d</code> =&gt; <code class="hljs">d, a, b, c</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the fourth value, and push it to the top</td>
</tr>
<tr>
<td>20</td>
<td><strong>rot5l</strong></td>
<td></td>
<td><code class="hljs">a, b, c, d, e</code> =&gt; <code class="hljs">e, a, b, c, d</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the fifth value, and push it to the top</td>
</tr>
<tr>
<td>21</td>
<td><strong>call_constructor</strong></td>
<td>2:argc</td>
<td><code class="hljs">new_target, func_obj, args...</code>
                =&gt; <code class="hljs">func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the constructor target value, function object and
                its argments from stack, call object constructor, push constructor's return value to
                the stack</td>
</tr>
<tr>
<td>22</td>
<td><strong>call</strong></td>
<td>2:argc</td>
<td><code class="hljs">func_obj, args...</code> =&gt; <code class="hljs">func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the function object and its argments from stack,
                call function, push function's return value to the stack</td>
</tr>
<tr>
<td>23</td>
<td><strong>tail_call</strong></td>
<td>2:argc</td>
<td><code class="hljs">func_obj, args...</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the function object and its argments from stack,
                call function, return the function's return value to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>24</td>
<td><strong>call_method</strong></td>
<td>2:argc</td>
<td><code class="hljs">func_obj, this, args...</code> =&gt; <code class="hljs">func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the function object, <code class="hljs">this</code>
                object, and its argments from stack, call method, push method's return value to the
                stack</td>
</tr>
<tr>
<td>25</td>
<td><strong>tail_call_method</strong></td>
<td>2:argc</td>
<td><code class="hljs">func_obj, this, args...</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the function object, <code class="hljs">this</code>
                object, and its argments from stack, call method, return the method's return value
                to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>26</td>
<td><strong>array_from</strong></td>
<td>2:argc</td>
<td><code class="hljs">args...</code> =&gt; <code class="hljs">
                [args]</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the function arguments from stack, push new array
                of arguments to the stack</td>
</tr>
<tr>
<td>27</td>
<td><strong>apply</strong></td>
<td>2:magic</td>
<td><code class="hljs">array[args], this, func_obj</code> =&gt; <code class="hljs">func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">call function or method with array of arguments, magic
                value: 0 = normal apply, 1 = apply for constructor, 2 = Reflect.apply, push return
                value of function to the stack</td>
</tr>
<tr>
<td>28</td>
<td><strong>return</strong></td>
<td></td>
<td><code class="hljs">func_ret</code> =&gt; <code class="hljs">
                .</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the function return value from stack, and return
                to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>29</td>
<td><strong>return_undef</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">return <code class="hljs">undefined</code> to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>2A</td>
<td><strong>check_ctor_return</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">is_ctor_return,
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">check a is <code class="hljs">undefined</code> or <code class="hljs">object</code>, derived class constructor must return an object or
                undefined</td>
</tr>
<tr>
<td>2B</td>
<td><strong>check_ctor</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">check calling function's argument, <code class="hljs">
                JSCallInternal(new_target)</code>, check <code class="hljs">new_target</code> != <code class="hljs">undefined</code>, class constructors must be invoked with 'new'</td>
</tr>
<tr>
<td>2C</td>
<td><strong>check_brand</strong></td>
<td></td>
<td><code class="hljs">func, obj</code> =&gt; <code class="hljs">func, obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">check home object of <code class="hljs">func</code> and <code class="hljs">obj</code> has same brand, if failed, exception returned to the
                caller to <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>2D</td>
<td><strong>add_brand</strong></td>
<td></td>
<td><code class="hljs">home_obj, obj</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">add a brand property to <code class="hljs">object</code>,
                which points to the brand of <code class="hljs">home_obj</code></td>
</tr>
<tr>
<td>2E</td>
<td><strong>return_async</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">save VM execution state, return <code class="hljs">
                undefined</code> to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>2F</td>
<td><strong>throw</strong></td>
<td></td>
<td><code class="hljs">except_obj</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the exception object from stack, set it as VM's
                current_exception</td>
</tr>
<tr>
<td>30</td>
<td><strong>throw_error</strong></td>
<td>4:atom, 1:type</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">throw an exception with type and error message atom</td>
</tr>
<tr>
<td>31</td>
<td><strong>eval</strong></td>
<td>2:argc, 2:scope</td>
<td><code class="hljs">eval_func, args...</code> =&gt; <code class="hljs">func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the <code class="hljs">eval</code> function object
                and its argments from stack, call function, push function's return value to the
                stack</td>
</tr>
<tr>
<td>32</td>
<td><strong>apply_eval</strong></td>
<td>2:scope</td>
<td><code class="hljs">array[args], eval_func_obj</code> =&gt; <code class="hljs">func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">take off the <code class="hljs">eval</code> function object
                and its argments from stack, call function, push function's return value to the
                stack</td>
</tr>
<tr>
<td>33</td>
<td><strong>regex</strong></td>
<td></td>
<td><code class="hljs">bytecode, pattern</code> =&gt; <code class="hljs">regexp_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">construct regular expression object from regexp bytecode
                and match pattern, push to the stack</td>
</tr>
<tr>
<td>34</td>
<td><strong>get_super</strong></td>
<td></td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">
                obj_super</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get object's prototype type object, replace top stack value</td>
</tr>
<tr>
<td>35</td>
<td><strong>import</strong></td>
<td></td>
<td><code class="hljs">specifier</code> =&gt; <code class="hljs">module</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">dynamic module import</td>
</tr>
<tr>
<td>36</td>
<td><strong>check_var</strong></td>
<td>4:atom</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">
                var_exists</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">check if a global variable exists</td>
</tr>
<tr>
<td>37</td>
<td><strong>get_var_undef</strong></td>
<td>4:atom</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get global variable and push to the stack, push <code class="hljs">undefined</code> if variable not exists</td>
</tr>
<tr>
<td>38</td>
<td><strong>get_var</strong></td>
<td>4:atom</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get global variable and push to the stack, throw if
                variable not exists</td>
</tr>
<tr>
<td>39</td>
<td><strong>put_var</strong></td>
<td>4:atom</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set global variable from stack stop, normal variable write,
                if variable not exist, throw</td>
</tr>
<tr>
<td>3A</td>
<td><strong>put_var_init</strong></td>
<td>4:atom</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set global variable from stack stop, if variable not exist,
                initialize lexical variable,</td>
</tr>
<tr>
<td>3B</td>
<td><strong>put_var_strict</strong></td>
<td>4:atom</td>
<td><code class="hljs">a, strict</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set global variable from stack stop, if <code class="hljs">
                strict</code> is true, throw if variable not exists, else normal variable write</td>
</tr>
<tr>
<td>3C</td>
<td><strong>get_ref_value</strong></td>
<td></td>
<td><code class="hljs">prop, this</code> =&gt; <code class="hljs">this[prop]</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get object's property</td>
</tr>
<tr>
<td>3D</td>
<td><strong>put_ref_value</strong></td>
<td></td>
<td><code class="hljs">a, prop, this</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set object's property, <code class="hljs">this[prop] = a</code>,
                if <code class="hljs">is_strict_mode()</code>, throw if property not exists</td>
</tr>
<tr>
<td>3E</td>
<td><strong>define_var</strong></td>
<td>4:atom, 1:flags</td>
<td><code class="hljs">.</code> =&gt; ``</td>
</tr>
<tr>
<td></td>
<td colspan="3">define global variable</td>
</tr>
<tr>
<td>3F</td>
<td><strong>check_define_var</strong></td>
<td>4:atom, 1:flags</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">check global variable re-declaration, flags is 0,
                DEFINE_GLOBAL_LEX_VAR or DEFINE_GLOBAL_FUNC_VAR</td>
</tr>
<tr>
<td>40</td>
<td><strong>define_func</strong></td>
<td>4:atom, 1:flags</td>
<td><code class="hljs">func_obj</code> =&gt; <code class="hljs">
                .</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">define global function</td>
</tr>
<tr>
<td>41</td>
<td><strong>get_field</strong></td>
<td>4:atom</td>
<td><code class="hljs">this</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">pop <code class="hljs">this</code> object, <code class="hljs">push this[prop]</code> to the stack</td>
</tr>
<tr>
<td>42</td>
<td><strong>get_field2</strong></td>
<td>4:atom</td>
<td><code class="hljs">this</code> =&gt; <code class="hljs">a,
                this</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push <code class="hljs">this[prop]</code> to the stack</td>
</tr>
<tr>
<td>43</td>
<td><strong>put_field</strong></td>
<td>4:atom</td>
<td><code class="hljs">a, this</code> =&gt; <code class="hljs">
                .</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set object property, <code class="hljs">this[prop] = a</code></td>
</tr>
<tr>
<td>44</td>
<td><strong>get_private_field</strong></td>
<td></td>
<td><code class="hljs">name, obj</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get object private property</td>
</tr>
<tr>
<td>45</td>
<td><strong>put_private_field</strong></td>
<td></td>
<td><code class="hljs">name, a, obj</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set object private property, <code class="hljs">obj[name] =
                a</code></td>
</tr>
<tr>
<td>46</td>
<td><strong>define_private_field</strong></td>
<td></td>
<td><code class="hljs">a, name, obj</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set object private property, <code class="hljs">obj[name] =
                a</code></td>
</tr>
<tr>
<td>47</td>
<td><strong>get_array_el</strong></td>
<td></td>
<td><code class="hljs">prop, this</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get array element, <code class="hljs">a = this[prop]</code></td>
</tr>
<tr>
<td>48</td>
<td><strong>get_array_el2</strong></td>
<td></td>
<td><code class="hljs">prop, this</code> =&gt; <code class="hljs">a, this</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get array element, <code class="hljs">a = this[prop]</code></td>
</tr>
<tr>
<td>49</td>
<td><strong>put_array_el</strong></td>
<td></td>
<td><code class="hljs">a, prop, this</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set array element, <code class="hljs">this[prop] = a</code></td>
</tr>
<tr>
<td>4A</td>
<td><strong>get_super_value</strong></td>
<td></td>
<td><code class="hljs">prop, obj, this</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get object property, <code class="hljs">Reflect.get(obj,
                prop, this)</code>, the value of <code class="hljs">this</code> provided for the <code class="hljs">obj</code> if a getter is encountered</td>
</tr>
<tr>
<td>4B</td>
<td><strong>put_super_value</strong></td>
<td></td>
<td><code class="hljs">a, prop, obj, this</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set object property, <code class="hljs">Reflect.set(obj,
                prop, a, this)</code>, the value of <code class="hljs">this</code> provided for the <code class="hljs">obj</code> if a setter is encountered</td>
</tr>
<tr>
<td>4C</td>
<td><strong>define_field</strong></td>
<td>4:atom</td>
<td><code class="hljs">a, this</code> =&gt; <code class="hljs">
                this</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">define object property, <code class="hljs">this[atom] = a</code></td>
</tr>
<tr>
<td>4D</td>
<td><strong>set_name</strong></td>
<td>4:atom</td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set the name of object</td>
</tr>
<tr>
<td>4E</td>
<td><strong>set_name_computed</strong></td>
<td></td>
<td><code class="hljs">obj, name</code> =&gt; <code class="hljs">obj, name</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set the name of object</td>
</tr>
<tr>
<td>4F</td>
<td><strong>set_proto</strong></td>
<td></td>
<td><code class="hljs">proto, obj</code> =&gt; <code class="hljs">obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set the prototype of object</td>
</tr>
<tr>
<td>50</td>
<td><strong>set_home_object</strong></td>
<td></td>
<td><code class="hljs">func_obj, home_obj</code> =&gt; <code class="hljs">func_obj, home_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set the home object of function</td>
</tr>
<tr>
<td>51</td>
<td><strong>define_array_el</strong></td>
<td></td>
<td><code class="hljs">a, prop, this</code> =&gt; <code class="hljs">prop, this</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">define array element, <code class="hljs">this[prop] = a</code></td>
</tr>
<tr>
<td>52</td>
<td><strong>append</strong></td>
<td></td>
<td><code class="hljs">enum_obj, pos, array</code> =&gt; <code class="hljs">pos, array</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">append to array, <code class="hljs">array[pos++] = a</code></td>
</tr>
<tr>
<td>53</td>
<td><strong>copy_data_properties</strong></td>
<td>1:mask</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">copy properties value from source to target object, stack
                offsets encoded in <code class="hljs">mask</code>: 2 bits for target, 3 bits for
                source, 2 bits for exclusionList</td>
</tr>
<tr>
<td>54</td>
<td><strong>define_method</strong></td>
<td>4:atom, 1:flags</td>
<td><code class="hljs">a, obj</code> =&gt; <code class="hljs">
                obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">define object method, <code class="hljs">obj[atom] = a</code></td>
</tr>
<tr>
<td>55</td>
<td><strong>define_method_computed</strong></td>
<td>1:flags</td>
<td><code class="hljs">a, atom, obj</code> =&gt; <code class="hljs">obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">define object method with computed method name, <code class="hljs">obj[atom] = a</code>, must come after <code class="hljs">
                define_method</code></td>
</tr>
<tr>
<td>56</td>
<td><strong>define_class</strong></td>
<td>4:atom, 1:flags</td>
<td><code class="hljs">class_ctor, parent</code> =&gt; <code class="hljs">proto, ctor</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">define a class object</td>
</tr>
<tr>
<td>57</td>
<td><strong>define_class_computed</strong></td>
<td>4:atom, 1:flags</td>
<td><code class="hljs">class_ctor, parent, name</code> =&gt; <code class="hljs">proto, ctor</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">define a class object with computed class name</td>
</tr>
<tr>
<td>58</td>
<td><strong>get_loc</strong></td>
<td>2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push local variable <code class="hljs">var_buf[index]</code>
                to the stack</td>
</tr>
<tr>
<td>59</td>
<td><strong>put_loc</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set local variable, <code class="hljs">var_buf[index] = a</code></td>
</tr>
<tr>
<td>5A</td>
<td><strong>set_loc</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set local variable, <code class="hljs">var_buf[index] = a</code></td>
</tr>
<tr>
<td>5B</td>
<td><strong>get_arg</strong></td>
<td>2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push function argument <code class="hljs">arg_buf[index]</code>
                to the stack</td>
</tr>
<tr>
<td>5C</td>
<td><strong>put_arg</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set function argument, <code class="hljs">arg_buf[index] =
                a</code></td>
</tr>
<tr>
<td>5D</td>
<td><strong>set_arg</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set function argument, <code class="hljs">arg_buf[index] =
                a</code></td>
</tr>
<tr>
<td>5E</td>
<td><strong>get_var_ref</strong></td>
<td>2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push variable reference <code class="hljs">var_refs[index]</code>
                to the stack</td>
</tr>
<tr>
<td>5F</td>
<td><strong>put_var_ref</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set variable reference, <code class="hljs">var_refs[index]
                = a</code></td>
</tr>
<tr>
<td>60</td>
<td><strong>set_var_ref</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set variable reference, <code class="hljs">var_refs[index]
                = a</code></td>
</tr>
<tr>
<td>61</td>
<td><strong>set_loc_uninitialized</strong></td>
<td>2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set local variable to <code class="hljs">JS_UNINITIALIZED</code>
                , <code class="hljs">var_buf[index] = uninitialized</code></td>
</tr>
<tr>
<td>62</td>
<td><strong>get_loc_check</strong></td>
<td>2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3"><code class="hljs">get_loc</code> with check for <code class="hljs">var_buf[index] != JS_UNINITIALIZED</code></td>
</tr>
<tr>
<td>63</td>
<td><strong>put_loc_check</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3"><code class="hljs">put_loc</code> with check for <code class="hljs">var_buf[index] != JS_UNINITIALIZED</code></td>
</tr>
<tr>
<td>64</td>
<td><strong>put_loc_check_init</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3"><code class="hljs">put_loc</code> with check for <code class="hljs">var_buf[index] == JS_UNINITIALIZED</code></td>
</tr>
<tr>
<td>65</td>
<td><strong>get_var_ref_check</strong></td>
<td>2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3"><code class="hljs">get_var_ref</code> with check for <code class="hljs">var_refs[index] != JS_UNINITIALIZED</code></td>
</tr>
<tr>
<td>66</td>
<td><strong>put_var_ref_check</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3"><code class="hljs">put_var_ref</code> with check for <code class="hljs">var_refs[index] != JS_UNINITIALIZED</code></td>
</tr>
<tr>
<td>67</td>
<td><strong>put_var_ref_check_init</strong></td>
<td>2:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3"><code class="hljs">put_var_ref</code> with check for <code class="hljs">var_refs[index] == JS_UNINITIALIZED</code></td>
</tr>
<tr>
<td>68</td>
<td><strong>close_loc</strong></td>
<td>2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">when leave scope, check to detach current stack frame's
                variable</td>
</tr>
<tr>
<td>69</td>
<td><strong>if_false</strong></td>
<td>4:label</td>
<td><code class="hljs">cond</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">offset current instruction pointer if false, <code class="hljs">if (!cond) pc += label</code></td>
</tr>
<tr>
<td>6A</td>
<td><strong>if_true</strong></td>
<td>4:label</td>
<td><code class="hljs">cond</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">offset current instruction pointer if true, <code class="hljs">if (cond) pc += label</code></td>
</tr>
<tr>
<td>6B</td>
<td><strong>goto</strong></td>
<td>4:label</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">offset current instruction pointer, <code class="hljs">pc
                += label</code></td>
</tr>
<tr>
<td>6C</td>
<td><strong>catch</strong></td>
<td>4:label</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">
                catch_offset</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push a catch_offset JSValue to stack, <code class="hljs">
                label</code> is the offset from the start of current function's byte_code_buf</td>
</tr>
<tr>
<td>6D</td>
<td><strong>gosub</strong></td>
<td>4:label</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">
                catch_offset</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push a catch_offset JSValue to stack, <code class="hljs">
                label</code> is the offset from the start of current function's byte_code_buf, and
                offset current instruction pointer, <code class="hljs">pc += label</code>, used to
                execute the finally block</td>
</tr>
<tr>
<td>6E</td>
<td><strong>ret</strong></td>
<td></td>
<td><code class="hljs">label</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set current instruction pointer, <code class="hljs">pc =
                byte_code_buf + label</code>, used to return from the finally block</td>
</tr>
<tr>
<td>6F</td>
<td><strong>to_object</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">obj_a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">convert top to stack to object</td>
</tr>
<tr>
<td>70</td>
<td><strong>to_propkey</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">
                propkey_a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">convert top to stack to property key, aka int, string, or
                symbol</td>
</tr>
<tr>
<td>71</td>
<td><strong>to_propkey2</strong></td>
<td></td>
<td><code class="hljs">a, obj</code> =&gt; <code class="hljs">propkey_a,
                obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">test <code class="hljs">obj != undefined &amp;&amp; obj !=
                null</code>, convert top to stack to property key, aka int, string, or symbol</td>
</tr>
<tr>
<td>72</td>
<td><strong>with_get_var</strong></td>
<td>4:atom, 4:label, 1:is_with</td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">
                new_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">replace stack top with <code class="hljs">obj[atom]</code>,
                offset current instruction pointer, <code class="hljs">pc += label - 5</code></td>
</tr>
<tr>
<td>73</td>
<td><strong>with_put_var</strong></td>
<td>4:atom, 4:label, 1:is_with</td>
<td><code class="hljs">obj, a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set object property, <code class="hljs">obj[atom] = a</code>,
                offset current instruction pointer, <code class="hljs">pc += label - 5</code></td>
</tr>
<tr>
<td>74</td>
<td><strong>with_delete_var</strong></td>
<td>4:atom, 4:label, 1:is_with</td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">
                success</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">delete object property, and push the delete status to
                stack, offset current instruction pointer, <code class="hljs">pc += label - 5</code></td>
</tr>
<tr>
<td>75</td>
<td><strong>with_make_ref</strong></td>
<td>4:atom, 4:label, 1:is_with</td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">atom,
                obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push the property name to stack, offset current instruction
                pointer, <code class="hljs">pc += label - 5</code></td>
</tr>
<tr>
<td>76</td>
<td><strong>with_get_ref</strong></td>
<td>4:atom, 4:label, 1:is_with</td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">a,
                obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get object property, <code class="hljs">a = obj[atom]</code>,
                offset current instruction pointer, <code class="hljs">pc += label - 5</code></td>
</tr>
<tr>
<td>77</td>
<td><strong>with_get_ref_undef</strong></td>
<td>4:atom, 4:label, 1:is_with</td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">a,
                undefined</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">get object property, <code class="hljs">a = obj[atom]</code>,
                set old stack top to <code class="hljs">undefined</code>, offset current instruction
                pointer, <code class="hljs">pc += label - 5</code></td>
</tr>
<tr>
<td>78</td>
<td><strong>make_loc_ref</strong></td>
<td>4:atom, 2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">make a new object, and a new local variable reference, set <code class="hljs">object[atom] = get_var_ref(index)</code></td>
</tr>
<tr>
<td>79</td>
<td><strong>make_arg_ref</strong></td>
<td>4:atom, 2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">make a new object, and a new function argument reference,
                set <code class="hljs">object[atom] = get_var_ref(index)</code></td>
</tr>
<tr>
<td>7A</td>
<td><strong>make_var_ref_ref</strong></td>
<td>4:atom, 2:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">make a new object, and a free variable reference, set <code class="hljs">object[atom] = var_refs[index]</code></td>
</tr>
<tr>
<td>7B</td>
<td><strong>make_var_ref</strong></td>
<td>4:atom</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">atom,
                global_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">construct a reference to a global variable</td>
</tr>
<tr>
<td>7C</td>
<td><strong>for_in_start</strong></td>
<td></td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">
                enum_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">construct <code class="hljs">for in</code> iterator from
                object</td>
</tr>
<tr>
<td>7D</td>
<td><strong>for_of_start</strong></td>
<td></td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">catch_offset,
                method, enum_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">construct <code class="hljs">for of</code> iterator from
                object</td>
</tr>
<tr>
<td>7E</td>
<td><strong>for_await_of_start</strong></td>
<td></td>
<td><code class="hljs">obj</code> =&gt; <code class="hljs">catch_offset,
                method, enum_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">construct <code class="hljs">for await of</code> iterator
                from object</td>
</tr>
<tr>
<td>7F</td>
<td><strong>for_in_next</strong></td>
<td></td>
<td><code class="hljs">enum_obj</code> =&gt; <code class="hljs">done,
                value, enum_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">iterator to the next value</td>
</tr>
<tr>
<td>80</td>
<td><strong>for_of_next</strong></td>
<td>1:index</td>
<td><code class="hljs">catch_offset, method, enum_obj</code>
                =&gt; <code class="hljs">done, value, catch_offset, method, enum_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">iterator to the next value</td>
</tr>
<tr>
<td>81</td>
<td><strong>iterator_check_object</strong></td>
<td></td>
<td><code class="hljs">iter_ret</code> =&gt; <code class="hljs">
                iter_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">check iterator must return an object</td>
</tr>
<tr>
<td>82</td>
<td><strong>iterator_get_value_done</strong></td>
<td></td>
<td><code class="hljs">iter_ret</code> =&gt; <code class="hljs">done,
                comp_val</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">check iterator must return an object, fetch the iterator
                complete value, and iterator done state</td>
</tr>
<tr>
<td>83</td>
<td><strong>iterator_close</strong></td>
<td></td>
<td><code class="hljs">catch_offset, method, enum_obj</code>
                =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">finalize iterator and cleanup</td>
</tr>
<tr>
<td>84</td>
<td><strong>iterator_close_return</strong></td>
<td></td>
<td><code class="hljs">ret_val, ..., catch_offset, method,
                enum_obj</code> =&gt; <code class="hljs">catch_offset, method, enum_obj, ret_val</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">finalize iterator and cleanup</td>
</tr>
<tr>
<td>85</td>
<td><strong>iterator_next</strong></td>
<td></td>
<td><code class="hljs">a, catch_offset, method, enum_obj</code>
                =&gt; <code class="hljs">b, catch_offset, method, enum_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">call iterator next method, <code class="hljs">b =
                iter_obj.method(a)</code></td>
</tr>
<tr>
<td>86</td>
<td><strong>iterator_call</strong></td>
<td>1:flags</td>
<td><code class="hljs">a, catch_offset, method, enum_obj</code>
                =&gt; <code class="hljs">ret_flag, b, catch_offset, method, enum_obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">call iterator next method, <code class="hljs">b =
                iter_obj.method(a)</code>, set <code class="hljs">ret_flag</code> in accordance with <code class="hljs">flags</code></td>
</tr>
<tr>
<td>87</td>
<td><strong>initial_yield</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">save VM execution state, return <code class="hljs">
                undefined</code> to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>88</td>
<td><strong>yield</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">save VM execution state, return <code class="hljs">
                FUNC_RET_YIELD</code> to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>89</td>
<td><strong>yield_star</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">save VM execution state, return <code class="hljs">
                FUNC_RET_YIELD_STAR</code> to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>8A</td>
<td><strong>async_yield_star</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">save VM execution state, return <code class="hljs">
                FUNC_RET_YIELD_STAR</code> to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>8B</td>
<td><strong>await</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">save VM execution state, return <code class="hljs">
                FUNC_RET_AWAIT</code> to the caller of <code class="hljs">JS_CallInternal()</code></td>
</tr>
<tr>
<td>8C</td>
<td><strong>neg</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">-a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">unary negative operator</td>
</tr>
<tr>
<td>8D</td>
<td><strong>plus</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">+a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">unary plus operator</td>
</tr>
<tr>
<td>8E</td>
<td><strong>dec</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a - 1</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">unary decrement operator</td>
</tr>
<tr>
<td>8F</td>
<td><strong>inc</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a + 1</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">unary increment operator</td>
</tr>
<tr>
<td>90</td>
<td><strong>post_dec</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a - 1,
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push <code class="hljs">a - 1</code> to the stack</td>
</tr>
<tr>
<td>91</td>
<td><strong>post_inc</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a + 1,
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">push <code class="hljs">a + 1</code> to the stack</td>
</tr>
<tr>
<td>92</td>
<td><strong>dec_loc</strong></td>
<td>1:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">decrement local variable, <code class="hljs">var_buf[index]
                = var_buf[index] - 1</code></td>
</tr>
<tr>
<td>93</td>
<td><strong>inc_loc</strong></td>
<td>1:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">increment local variable, <code class="hljs">var_buf[index]
                = var_buf[index] + 1</code></td>
</tr>
<tr>
<td>94</td>
<td><strong>add_loc</strong></td>
<td>1:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">add <code class="hljs">a</code> to local variable, <code class="hljs">var_buf[index] = var_buf[index] + a</code></td>
</tr>
<tr>
<td>95</td>
<td><strong>not</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">~a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">binary not operator</td>
</tr>
<tr>
<td>96</td>
<td><strong>lnot</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">!a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">boolean not operator</td>
</tr>
<tr>
<td>97</td>
<td><strong>typeof</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">typeof
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3"><code class="hljs">typeof</code> operator</td>
</tr>
<tr>
<td>98</td>
<td><strong>delete</strong></td>
<td></td>
<td><code class="hljs">prop, obj</code> =&gt; <code class="hljs">success</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">delete object property, return whether delete succeeds</td>
</tr>
<tr>
<td>99</td>
<td><strong>delete_var</strong></td>
<td>4:atom</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">
                success</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">delete global object's property, return whether delete
                succeeds</td>
</tr>
<tr>
<td>9A</td>
<td><strong>mul</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b *
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">multiply operator</td>
</tr>
<tr>
<td>9B</td>
<td><strong>div</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b /
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">divide operator</td>
</tr>
<tr>
<td>9C</td>
<td><strong>mod</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b %
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">modulate operator</td>
</tr>
<tr>
<td>9D</td>
<td><strong>add</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b +
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">add operator</td>
</tr>
<tr>
<td>9E</td>
<td><strong>sub</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b -
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">subtraction operator</td>
</tr>
<tr>
<td>9F</td>
<td><strong>pow</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">pow(b,
                a)</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">power operator</td>
</tr>
<tr>
<td>A0</td>
<td><strong>shl</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                &lt;&lt; a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">shift logical left operator</td>
</tr>
<tr>
<td>A1</td>
<td><strong>sar</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                &gt;&gt; a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">shift arithmetic right operator</td>
</tr>
<tr>
<td>A2</td>
<td><strong>shr</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                &gt;&gt; a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">shift logical right operator</td>
</tr>
<tr>
<td>A3</td>
<td><strong>lt</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                &lt; a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">less than compare</td>
</tr>
<tr>
<td>A4</td>
<td><strong>lte</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                &lt;= a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">less than or equal compare</td>
</tr>
<tr>
<td>A5</td>
<td><strong>gt</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                &gt; a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">greater compare</td>
</tr>
<tr>
<td>A6</td>
<td><strong>gte</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                &gt;= a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">greater or equal compare</td>
</tr>
<tr>
<td>A7</td>
<td><strong>instanceof</strong></td>
<td></td>
<td><code class="hljs">obj, cls</code> =&gt; <code class="hljs">obj
                instanceof cls</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">object is instanceof class</td>
</tr>
<tr>
<td>A8</td>
<td><strong>in</strong></td>
<td></td>
<td><code class="hljs">obj, prop</code> =&gt; <code class="hljs">prop in obj</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">check object has property</td>
</tr>
<tr>
<td>A9</td>
<td><strong>eq</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                == a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">equal compare</td>
</tr>
<tr>
<td>AA</td>
<td><strong>neq</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                != a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">not equal compare</td>
</tr>
<tr>
<td>AB</td>
<td><strong>strict_eq</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                === a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">strict equal compare</td>
</tr>
<tr>
<td>AC</td>
<td><strong>strict_neq</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                !== a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">not strict equal compare</td>
</tr>
<tr>
<td>AD</td>
<td><strong>and</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b
                &amp; a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">binary and operator</td>
</tr>
<tr>
<td>AE</td>
<td><strong>xor</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b ^
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">binary xor operator</td>
</tr>
<tr>
<td>AF</td>
<td><strong>or</strong></td>
<td></td>
<td><code class="hljs">a, b</code> =&gt; <code class="hljs">b |
                a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">binary or operator</td>
</tr>
<tr>
<td>B0</td>
<td><strong>is_undefined_or_null</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">
                a_is_undefined_or_null</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">check <code class="hljs">a === undefined || a === null</code></td>
</tr>
<tr>
<td></td>
<td><strong></strong></td>
<td></td>
<td></td>
</tr>
<tr>
<td></td>
<td colspan="3"></td>
</tr>
<tr>
<td></td>
<td><strong>SHORT OPCODES</strong></td>
<td></td>
<td></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcodes</td>
</tr>
<tr>
<td></td>
<td><strong></strong></td>
<td></td>
<td></td>
</tr>
<tr>
<td></td>
<td colspan="3"></td>
</tr>
<tr>
<td>B1</td>
<td><strong>nop</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">no-operation, VM will skip this opcode</td>
</tr>
<tr>
<td>B2</td>
<td><strong>push_minus1</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = -1</code></td>
</tr>
<tr>
<td>B3</td>
<td><strong>push_0</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = 0</code></td>
</tr>
<tr>
<td>B4</td>
<td><strong>push_1</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = 1</code></td>
</tr>
<tr>
<td>B5</td>
<td><strong>push_2</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = 2</code></td>
</tr>
<tr>
<td>B6</td>
<td><strong>push_3</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = 3</code></td>
</tr>
<tr>
<td>B7</td>
<td><strong>push_4</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = 4</code></td>
</tr>
<tr>
<td>B8</td>
<td><strong>push_5</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = 5</code></td>
</tr>
<tr>
<td>B9</td>
<td><strong>push_6</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = 6</code></td>
</tr>
<tr>
<td>BA</td>
<td><strong>push_7</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = 7</code></td>
</tr>
<tr>
<td>BB</td>
<td><strong>push_i8</strong></td>
<td>1:value</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = value</code></td>
</tr>
<tr>
<td>BC</td>
<td><strong>push_i16</strong></td>
<td>2:value</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = value</code></td>
</tr>
<tr>
<td>BD</td>
<td><strong>push_const8</strong></td>
<td>1:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push <code class="hljs">a = cpool[index]</code></td>
</tr>
<tr>
<td>BE</td>
<td><strong>fclosure8</strong></td>
<td>1:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, fetch function from the constant pool, bind
                var refs with current context, push the new closure</td>
</tr>
<tr>
<td>BF</td>
<td><strong>push_empty_string</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push empty string</td>
</tr>
<tr>
<td>C0</td>
<td><strong>get_loc8</strong></td>
<td>1:index</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push local variable <code class="hljs">
                var_buf[index]</code> to the stack</td>
</tr>
<tr>
<td>C1</td>
<td><strong>put_loc8</strong></td>
<td>1:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[index]
                = a</code></td>
</tr>
<tr>
<td>C2</td>
<td><strong>set_loc8</strong></td>
<td>1:index</td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[index]
                = a</code></td>
</tr>
<tr>
<td>C3</td>
<td><strong>get_loc0</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push local variable <code class="hljs">
                var_buf[0]</code> to the stack</td>
</tr>
<tr>
<td>C4</td>
<td><strong>get_loc1</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push local variable <code class="hljs">
                var_buf[1]</code> to the stack</td>
</tr>
<tr>
<td>C5</td>
<td><strong>get_loc2</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push local variable <code class="hljs">
                var_buf[2]</code> to the stack</td>
</tr>
<tr>
<td>C6</td>
<td><strong>get_loc3</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push local variable <code class="hljs">
                var_buf[3]</code> to the stack</td>
</tr>
<tr>
<td>C7</td>
<td><strong>put_loc0</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[0]
                = a</code></td>
</tr>
<tr>
<td>C8</td>
<td><strong>put_loc1</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[1]
                = a</code></td>
</tr>
<tr>
<td>C9</td>
<td><strong>put_loc2</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[2]
                = a</code></td>
</tr>
<tr>
<td>CA</td>
<td><strong>put_loc3</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[3]
                = a</code></td>
</tr>
<tr>
<td>CB</td>
<td><strong>set_loc0</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[0]
                = a</code></td>
</tr>
<tr>
<td>CC</td>
<td><strong>set_loc1</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[1]
                = a</code></td>
</tr>
<tr>
<td>CD</td>
<td><strong>set_loc2</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[2]
                = a</code></td>
</tr>
<tr>
<td>CE</td>
<td><strong>set_loc3</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set local variable, <code class="hljs">var_buf[3]
                = a</code></td>
</tr>
<tr>
<td>CF</td>
<td><strong>get_arg0</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push function argument <code class="hljs">
                arg_buf[0]</code> to the stack</td>
</tr>
<tr>
<td>D0</td>
<td><strong>get_arg1</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push function argument <code class="hljs">
                arg_buf[1]</code> to the stack</td>
</tr>
<tr>
<td>D1</td>
<td><strong>get_arg2</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push function argument <code class="hljs">
                arg_buf[2]</code> to the stack</td>
</tr>
<tr>
<td>D2</td>
<td><strong>get_arg3</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push function argument <code class="hljs">
                arg_buf[3]</code> to the stack</td>
</tr>
<tr>
<td>D3</td>
<td><strong>put_arg0</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set function argument, <code class="hljs">arg_buf[0]
                = a</code></td>
</tr>
<tr>
<td>D4</td>
<td><strong>put_arg1</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set function argument, <code class="hljs">arg_buf[1]
                = a</code></td>
</tr>
<tr>
<td>D5</td>
<td><strong>put_arg2</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set function argument, <code class="hljs">arg_buf[2]
                = a</code></td>
</tr>
<tr>
<td>D6</td>
<td><strong>put_arg3</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set function argument, <code class="hljs">arg_buf[3]
                = a</code></td>
</tr>
<tr>
<td>D7</td>
<td><strong>set_arg0</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set function argument, <code class="hljs">arg_buf[0]
                = a</code></td>
</tr>
<tr>
<td>D8</td>
<td><strong>set_arg1</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set function argument, <code class="hljs">arg_buf[1]
                = a</code></td>
</tr>
<tr>
<td>D9</td>
<td><strong>set_arg2</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set function argument, <code class="hljs">arg_buf[2]
                = a</code></td>
</tr>
<tr>
<td>DA</td>
<td><strong>set_arg3</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set function argument, <code class="hljs">arg_buf[3]
                = a</code></td>
</tr>
<tr>
<td>DB</td>
<td><strong>get_var_ref0</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push variable reference <code class="hljs">
                var_refs[0]</code> to the stack</td>
</tr>
<tr>
<td>DC</td>
<td><strong>get_var_ref1</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push variable reference <code class="hljs">
                var_refs[1]</code> to the stack</td>
</tr>
<tr>
<td>DD</td>
<td><strong>get_var_ref2</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push variable reference <code class="hljs">
                var_refs[2]</code> to the stack</td>
</tr>
<tr>
<td>DE</td>
<td><strong>get_var_ref3</strong></td>
<td></td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, push variable reference <code class="hljs">
                var_refs[3]</code> to the stack</td>
</tr>
<tr>
<td>DF</td>
<td><strong>put_var_ref0</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set variable reference, <code class="hljs">var_refs[0]
                = a</code></td>
</tr>
<tr>
<td>E0</td>
<td><strong>put_var_ref1</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set variable reference, <code class="hljs">var_refs[1]
                = a</code></td>
</tr>
<tr>
<td>E1</td>
<td><strong>put_var_ref2</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set variable reference, <code class="hljs">var_refs[2]
                = a</code></td>
</tr>
<tr>
<td>E2</td>
<td><strong>put_var_ref3</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set variable reference, <code class="hljs">var_refs[3]
                = a</code></td>
</tr>
<tr>
<td>E3</td>
<td><strong>set_var_ref0</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set variable reference, <code class="hljs">var_refs[0]
                = a</code></td>
</tr>
<tr>
<td>E4</td>
<td><strong>set_var_ref1</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set variable reference, <code class="hljs">var_refs[1]
                = a</code></td>
</tr>
<tr>
<td>E5</td>
<td><strong>set_var_ref2</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set variable reference, <code class="hljs">var_refs[2]
                = a</code></td>
</tr>
<tr>
<td>E6</td>
<td><strong>set_var_ref3</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, set variable reference, <code class="hljs">var_refs[3]
                = a</code></td>
</tr>
<tr>
<td>E7</td>
<td><strong>get_length</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">
                length(a)</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, get length of stack top value</td>
</tr>
<tr>
<td>E8</td>
<td><strong>if_false8</strong></td>
<td>1:label</td>
<td><code class="hljs">cond</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, offset current instruction pointer if false, <code class="hljs">if (!cond) pc += label</code></td>
</tr>
<tr>
<td>E9</td>
<td><strong>if_true8</strong></td>
<td>1:label</td>
<td><code class="hljs">cond</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, offset current instruction pointer if false, <code class="hljs">if (cond) pc += label</code></td>
</tr>
<tr>
<td>EA</td>
<td><strong>goto8</strong></td>
<td>1:label</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, offset current instruction pointer, <code class="hljs">pc += label</code></td>
</tr>
<tr>
<td>EB</td>
<td><strong>goto16</strong></td>
<td>2:label</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, offset current instruction pointer, <code class="hljs">pc += label</code></td>
</tr>
<tr>
<td>EC</td>
<td><strong>call0</strong></td>
<td></td>
<td><code class="hljs">func_obj</code> =&gt; <code class="hljs">
                func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, for <code class="hljs">call</code> with argc
                = 0</td>
</tr>
<tr>
<td>ED</td>
<td><strong>call1</strong></td>
<td></td>
<td><code class="hljs">func_obj, arg0</code> =&gt; <code class="hljs">func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, for <code class="hljs">call</code> with argc
                = 1</td>
</tr>
<tr>
<td>EE</td>
<td><strong>call2</strong></td>
<td></td>
<td><code class="hljs">func_obj, arg0, arg1</code> =&gt; <code class="hljs">func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, for <code class="hljs">call</code> with argc
                = 2</td>
</tr>
<tr>
<td>EF</td>
<td><strong>call3</strong></td>
<td></td>
<td><code class="hljs">func_obj, arg0, arg1, arg2</code> =&gt; <code class="hljs">func_ret</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, for <code class="hljs">call</code> with argc
                = 3</td>
</tr>
<tr>
<td>F0</td>
<td><strong>is_undefined</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a ===
                undefined</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, check <code class="hljs">a === undefined</code></td>
</tr>
<tr>
<td>F1</td>
<td><strong>is_null</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">a ===
                null</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, check <code class="hljs">a === undefined</code></td>
</tr>
<tr>
<td>F2</td>
<td><strong>typeof_is_undefined</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">typeof
                a === 'undefined'</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, check <code class="hljs">typeof a ===
                'undefined'</code></td>
</tr>
<tr>
<td>F3</td>
<td><strong>typeof_is_function</strong></td>
<td></td>
<td><code class="hljs">a</code> =&gt; <code class="hljs">typeof
                a === 'function'</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">short opcode, check <code class="hljs">typeof a ===
                'function'</code></td>
</tr>
<tr>
<td></td>
<td><strong></strong></td>
<td></td>
<td></td>
</tr>
<tr>
<td></td>
<td colspan="3"></td>
</tr>
<tr>
<td></td>
<td><strong>TEMPORARY OPCODES</strong></td>
<td></td>
<td></td>
</tr>
<tr>
<td></td>
<td colspan="3">temporary opcodes, removed later</td>
</tr>
<tr>
<td></td>
<td><strong></strong></td>
<td></td>
<td></td>
</tr>
<tr>
<td></td>
<td colspan="3"></td>
</tr>
<tr>
<td>B2</td>
<td><strong>enter_scope</strong></td>
<td>2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>B3</td>
<td><strong>leave_scope</strong></td>
<td>2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>B4</td>
<td><strong>label</strong></td>
<td>4:label</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_labels()</code></td>
</tr>
<tr>
<td>B5</td>
<td><strong>scope_get_var_undef</strong></td>
<td>4:var_name, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>B6</td>
<td><strong>scope_get_var</strong></td>
<td>4:var_name, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>B7</td>
<td><strong>scope_put_var</strong></td>
<td>4:var_name, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>B8</td>
<td><strong>scope_delete_var</strong></td>
<td>4:var_name, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>B9</td>
<td><strong>scope_make_ref</strong></td>
<td>4:var_name, 4:label, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>BA</td>
<td><strong>scope_get_ref</strong></td>
<td>4:var_name, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>BB</td>
<td><strong>scope_put_var_init</strong></td>
<td>4:var_name, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>BC</td>
<td><strong>scope_get_private_field</strong></td>
<td>4:var_name, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>BD</td>
<td><strong>scope_get_private_field2</strong></td>
<td>4:var_name, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>BE</td>
<td><strong>scope_put_private_field</strong></td>
<td>4:var_name, 2:scope</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">removed later in <code class="hljs">resolve_variables()</code></td>
</tr>
<tr>
<td>BF</td>
<td><strong>set_class_name</strong></td>
<td>4:class_offset</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">set class name</td>
</tr>
<tr>
<td>C0</td>
<td><strong>line_num</strong></td>
<td>4:line</td>
<td><code class="hljs">.</code> =&gt; <code class="hljs">.</code></td>
</tr>
<tr>
<td></td>
<td colspan="3">debug line number</td>
</tr>
</tbody>
</table>

