# @thi.ng/union-struct

[![npm](https://img.shields.io/npm/v/@thi.ng/unionstruct.svg)](https://www.npmjs.com/package/@thi.ng/unionstruct)

## Overview

C-style struct, union and bitfield read/write views of ArrayBuffers. Written in
TypeScript, distributed as ES6.

Features:

- construct memory mapped JS objects based on given typedef specs
- nested structs & unions
- packed bitfields (signed / unsigned)
- auto-alignment of fields to respective word boundaries (can be disabled)
- configurable endianess (bitfields currently assume network order / big endian)
- no runtime dependencies, works in node & browser
- small: 2.35KB minified, 1.14KB gzipped

Currently does not support array fields (incl. strings).

## Installation

```
yarn add @thi.ng/unionstruct
```

## API

```js
let {struct, union, sizeOf} = require("@thi.ng/unionstruct");
```

C-style union types define alternate views of the same data. For example this C
snippet below defines such a type, of which the first 32-bits can be accessed
either via individual bitfields or as combined value. Fields in this union type
can be accessed like `x.flags` (combined) or `x.state.cache` (only bits 9-11 of
`x.flags`). Since all views share the same memory, value changes of one
view are reflected in all others too (of course updating bitfields only modifies
a field's allocated bit range).

```c
// C
typedef union {
    uint32_t flags;
    struct {
        uint32_t type : 9;
        uint32_t cache : 3;
        uint32_t enabled : 1;
        uint32_t visible : 1;
        uint32_t selected : 1;
        uint32_t locked : 1;
        int32_t scheduled : 18;
        uint16_t tag;
    } state;
} Header;
```

This library provides this similarly in JS. The field spec format,
bitfields and alignment are described further below.

```js
typedef_header = [
    ["flags", "u32"],
    ["state", "struct", [
        ["type", "u32", 9],
        ["cache", "u32", 3],
        ["enabled", "u32", 1],
        ["visible", "u32", 1],
        ["selected", "u32", 1],
        ["locked", "u32", 1],
        ["scheduled", "i32", 18],
        ["tag", "u16"]]]
];

// pre-loaded binary data
buf = new Uint32Array([0x807cc0, 0x40000000, 0x3930]);

// create instance, see list of header arguments below
header = union(typedef_header, buf.buffer);
// { __buffer: ArrayBuffer { byteLength: 12 },
//   __spec: [['flags', 'u32'], ['state', 'struct', [...]]],
//   __size: 80,
//   __offsets: { flags: 0, state: 0 },
//   flags: [Getter/Setter],
//   state: [Getter] }

header.state
// { __buffer: ArrayBuffer { byteLength: 12 },
//   __spec: [...],
//   __size: 80,
//   __offsets:
//    { type: 0,
//      cache: 9,
//      enabled: 12,
//      visible: 13,
//      selected: 14,
//      locked: 15,
//      scheduled: 16,
//      tag: 64 },
// ... }

header.flags.toString(16) // "c07c8000"
header.state.type         // 384
header.state.enabled      // 1
header.state.visible      // 1
header.state.selected     // 0
header.state.locked       // 0
header.state.scheduled    // -131072
header.state.tag          // 12345
```

### union(spec: Field[], buf?: ArrayBuffer, offset = 0, align = true, littleEndian = false) => any

Takes an array of field specs (as in example above) and optional ArrayBuffer,
offset etc. If no buffer is given, constructs a new one with minimum size
required by this field spec. Returns an object with enumerable field accessors
and the following additional keys (largely for introspection purposes):

- `__buffer` - backing ArrayBuffer instance
- `__offsets` - **bit offset** in buffer for each field
- `__spec` - original field spec array provided
- `__size` - computed **bit size** of whole type

All top-level fields in a union share the same start address.
Also see note about [alignment](#alignment) below.

### struct(spec: Field[], buf?: ArrayBuffer, offset = 0, align = true, littleEndian = false) => any

Same as `union`, but field start addresses are arranged sequentially (and aligned individually).

### sizeOf(spec: Field[], union = false, doAlign = true) => number

Returns bit size of given field spec, taking into account alignment.

```js
// struct
sizeOf([["a", "u32", 14], ["b", "u32", 6], ["c","u8"]]);
// 40

// union
sizeOf([["a", "u32", 14], ["b", "u32", 6], ["c","u8"]], true);
// 14

```
### Alignment

For unions, if `align` is enabled (default), the entire type's offset will be
aligned to the largest required width. E.g. If any of the top-level fields is
of type `f64`, alignment will be to 8-byte boundaries. If the union contains
nested types, they will be checked recursively and aligned to largest type
found (for structs only the first field has an impact on whole struct
alignment).

| Type      | Alignment |
|-----------|----------:|
| f64       | 8         |
| f32       | 4         |
| u32 / i32 | 4         |
| u16 / i16 | 2         |
| u8 / i8   | 1         |

### Bitfields

Bitfields can only use integer types and support both signed / unsigned
flavors. Successive bitfields are densely packed (no alignment in between). The
max. width of a single field is 32 bits, but an arbitrary number of successive
bitfields can be defined.

If `align` is enabled and the last bitfield in a group does not end at a word
boundary, the field will be padded invisibly, based on its type (has no impact
on size of last field).

```js
bitfields = struct([
    // 2 packed bitfields (20 bits)
    ["a", "u32", 14],
    ["b", "u32", 6],
    // 32 - 20 = 12 bit padding here
    ["c", "u8"] // providing no bit width forces alignment
]);

bitfields.__offsets
// { a: 0, b: 14, c: 32 }

// without padding, field `c` would incorrectly start at bit offset 24
// (since u8 aligns itself to 8-bit boundaries)
```

## Typescript specifics

The library defines a `FieldType` type alias and `Field` interface to allow for
type checking of field specs (details [here](https://github.com/thi-ng/unionstruct/blob/master/src/index.ts)).
These are exposed in the `index.d.ts` file, bundled in the release version.

# Authors

- Karsten Schmidt

# License

&copy; 2017 Karsten Schmidt // Apache Software License 2.0