# JSValue Representation

JSValue is a Javascript value which can be a primitive type (such as Number, String, ...) or an Object.

**kwui** is compiling QuickJS with strict nan-boxing, for the purpose of cross-platform compatibility. JSValue is 64-bit on 32-bit and 64-bit platforms.


## JSValue Format

QuickJS number is double-precision floating-point number. double type is 64-bit, comprised of 1 sign bit,
11 exponent bits and 52 mantissa bits.

```
   7         6        5        4        3        2        1        0
seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
```

JSValue store non-NaN float with all bits **_binary reversed_**, therefore the first 12 bits are non-zero.

NaN numbers, and other primitive types, are represented by `tag` and `value`.

```
00000000|0000tttt|vvvvvvvv|vvvvvvvv|vvvvvvvv|vvvvvvvv|vvvvvvvv|vvvvvvvv
12-bits zero |tag|  48-bit placeholder for values: pointers, strings
```

#
| Primitive Type | tag | value |
|:--|:--|:--|
NaN number  | 7 | 0
-Inf number | 7 | 1
+Inf number | 7 | 2
`int`       | 1 | as-is
`false`     | 2 | 0
`true`      | 2 | 1
`null`      | 3 | 0
`undefined` | 4 | 0
`exception` | 6 | 0
`object`    | 8 | `JSObject*`
`string`    | 11 | `JSString*`
`symbol`    | 12 | `JSAtomStruct*`

Reference counted JSValue type (such as object, string, symbol), the most significant bit of `tag` is 1.
