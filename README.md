# postcard-nv

Postcard is a compact binary format for typed values. A value is written
as its parts, one after another, with nothing between them: no field
names, no offsets, no framing. Both ends must already agree on the shape
of the data. The format is
[postcard](https://postcard.jamesmunns.com/wire-format), which is what
embedded Rust programs use for the payloads they send. This package
brings it to novo-lang.

## What the format is

A **varint** is an integer written seven bits per byte, low bits first,
with the top bit of each byte set while more bytes follow. This is the
LEB128 encoding. A small number costs one byte and a large one costs up
to ten.

A signed integer is **zigzag folded** before it is written: 0 stays 0,
-1 becomes 1, 1 becomes 2, -2 becomes 3, and so on. Folding is what
keeps a small negative number small, because sign extension would make
every negative number ten bytes.

Everything else follows from those two. A string is a varint byte count
and then the UTF-8 bytes. A run of bytes is a varint count and then the
bytes. A sequence is a varint element count and then the elements. An
enum value is a varint variant index and then the payload. An optional
value is one discriminant byte, `0x00` for nothing and `0x01` followed
by the value for something. A 64-bit float is eight little-endian bytes.
The unit value is nothing at all.

A struct is its members in declaration order, end to end. There is no
mark between them, so a reader recovers them only by knowing the type it
is reading.

| Item | Encoding |
| --- | --- |
| Unsigned integer | LEB128 varint, 1 to 10 bytes |
| Signed integer | zigzag, then LEB128 varint |
| One-byte integer | one raw byte, not varinted |
| Boolean | one byte, 0 or 1 |
| 64-bit float | 8 little-endian bytes |
| String, byte run | varint length, then the bytes |
| Sequence | varint element count, then the elements |
| Enum value | varint variant index, then the payload |
| Optional value | `0x00`, or `0x01` and the value |
| Unit | 0 bytes |
| Struct | its members, end to end |

## Install

```
novo pkg add postcard-nv
```

## Example

```novo
use std.bytes
use postcard

struct Reading
    sensor: Int
    celsius: Float

fn main() [io]
    // Nine bytes: 06 is the zigzag varint for 3, and the eight that
    // follow are 21.5 as a little-endian double.
    var b = postcard.to_bytes(Reading { sensor: 3, celsius: 21.5 })
    println(bytes.to_hex(b))   // 060000000000803540

    // Reading it back is done part by part, in the order written.
    var c = bytes.cursor_le(b)
    match postcard.take_signed(c)
        Ok(sensor) => println("sensor ${sensor}")   // sensor 3
        Err(e)     => println(e.message())
    match postcard.take_f64(c)
        Ok(t)  => println("celsius ${t}")           // celsius 21.5
        Err(e) => println(e.message())
```

`postcard::from_bytes::<Reading>` in Rust, with `sensor: i64` and
`celsius: f64`, reads the same nine bytes.

## What the package contains

| Module | Contents |
| --- | --- |
| `postcard` | The whole package: the two size questions, a `put_` and a `take_` function for every part of the format, the writer that carries the standard library's `Serializer`, and the five named refusals. |

## How to choose an entry point

**`postcard.to_bytes` writes a novo-lang value through the standard
library's `Serialize` trait.** Reach for it when the other end's integers
are signed or are written as novo-lang writes every `Int`. See rule 3.

**The `put_` functions write the format part by part over a
`Cursor`.** Reach for them when the other end is Rust and its struct
holds an unsigned integer, a `u8`, or a tuple.

**The `take_` functions read the format part by part over a
`Cursor`.** They are the way to read a document. See rule 5.

## The rules a user needs

1. **The two ends must agree on the type.** The bytes carry no names, no
   lengths above the ones listed in the table, and no framing. A
   document read as the wrong type produces wrong values and no
   diagnostic.
2. **There is no framing.** A reader of a stream needs a length prefix
   or a delimiter from somewhere else.
   [cobs-nv](https://novo-lang.org/packages/cobs-nv) and
   [frame-nv](https://novo-lang.org/packages/frame-nv) are those
   somewhere elses.
3. **novo-lang has one integer type and postcard has ten integer
   encodings.** Every `Int` written through the trait goes out as a
   zigzag varint, which is what Rust postcard does for `i16`, `i32`,
   `i64` and `isize`. A Rust member of type `u16`, `u32`, `u64` or
   `usize` is a plain LEB128 varint with no zigzag, and a `u8` or `i8`
   is one raw byte. Write those with `postcard.put_unsigned` and
   `postcard.put_byte`.
4. **An optional member is a discriminant and then the value.**
   `None` is `0x00`, and `Some(5)` is `0x01` and then `5`'s encoding.
   The writer writes both through the standard library's
   `begin_some` hook, and `postcard.take_option` reads the
   discriminant back.
5. **A document is read part by part, not through the trait.** The
   standard library's `Deserializer` answers a child cursor for each
   member and leaves the parent where it was. A format with names finds
   each member by its name; postcard has no names, so member 2 begins
   where member 1 ended and the parent never learns where that was. The
   `take_` functions read over a `Cursor`, which does advance, so a
   reader calls them in the order the members were written.
6. **An enum value is its variant index and then its payload**, under
   the default external tagging. An enum declared with internal or
   adjacent tagging has its tag written as a string as well, which Rust
   postcard does not do.
7. **A tuple is written with an element count in front.** The
   `Serializer` announces a tuple as a sequence, and postcard writes a
   Rust tuple with no count. For a Rust tuple, write the elements with
   the `put_` functions.
8. **An overlong varint is refused.** A varint longer than ten bytes, or
   whose tenth byte holds bits above bit 63, is
   `PostcardOverlongVarint`. A varint with excess zero groups inside
   ten bytes is accepted, as the specification's canonicalisation table
   says.
9. **A discriminant that is neither 0 nor 1 is refused.** That covers a
   boolean byte and an optional's tag, and it is `PostcardBadTag`.
10. **A string's bytes must be UTF-8**, as RFC 3629 defines it.
    Otherwise `PostcardBadUtf8`, with the offset of the string's first
    byte.
11. **Reading one value out of a larger buffer leaves the rest.** The
    cursor's position is where the next value starts.
    `postcard.take_end` answers `PostcardTrailingBytes` for a caller that
    expected the buffer to hold exactly one value.
12. **An `f64` is always little-endian.** `put_f64` and `take_f64` write
    and read the bytes one at a time, so the cursor's own byte order
    does not matter.

## What is not included

- **Reading through the `Deserialize` trait.** See rule 5. The
  standard library would need a cursor that advances through its
  `Deserializer` methods before a `from_bytes` could be written.
- **A build for a microcontroller.** The surface speaks `Bytes`, `Str`
  and `Result`, none of which links on a device today, so this package
  does not build for a microcontroller with no heap allocator and
  carries no probe program. The device packages beside it are
  [rzcobs-nv](https://novo-lang.org/packages/rzcobs-nv),
  [bbqueue-nv](https://novo-lang.org/packages/bbqueue-nv) and
  [heapless-nv](https://novo-lang.org/packages/heapless-nv).
- **Framing.** See rule 2.
- **Schema description of any kind.** A postcard document says nothing
  about itself. A self-describing payload wants
  [msgpack-nv](https://novo-lang.org/packages/msgpack-nv) or
  [cbor-nv](https://novo-lang.org/packages/cbor-nv).
- **The postcard-rpc vocabulary.** Endpoints, topics, a
  request-and-reply correlation key and a sequence number all sit on top
  of these bytes, so they belong to a package that depends on this one.
- **32-bit floats and 128-bit integers.** postcard writes `f32` as four
  little-endian bytes and `u128` as a varint of up to nineteen bytes,
  and novo-lang's `Float` and `Int` are 64-bit.
- **Any input or output.** Every function here is arithmetic over bytes
  the caller already holds.

## Related packages

- [leb128-nv](https://novo-lang.org/packages/leb128-nv) and
  [zigzag-nv](https://novo-lang.org/packages/zigzag-nv) are this
  package's two dependencies. Postcard's integers are those two
  encodings.
- [msgpack-nv](https://novo-lang.org/packages/msgpack-nv) and
  [cbor-nv](https://novo-lang.org/packages/cbor-nv) are the opposite
  trade: a tag in front of every value, so a reader needs no schema and
  the document is larger.
- [serde-nv](https://novo-lang.org/packages/serde-nv) is the
  serialisation framework and its own formats.
- [rzcobs-nv](https://novo-lang.org/packages/rzcobs-nv) and
  [cobs-nv](https://novo-lang.org/packages/cobs-nv) frame a payload for
  a serial link, which is what a postcard document usually travels in.
- [protobuf-nv](https://novo-lang.org/packages/protobuf-nv) also needs a
  schema, but tags each field with a number, so a reader can skip fields
  it does not know.

## Tests

```bash
novo test tests/postcard_tests.nv      # 34 tests
bash tests/coverage.sh                 # line coverage over src/
```

The vectors are the postcard wire format specification's: the unsigned
and signed varint tables of "varint encoded integers", the
canonicalisation table, and the `f64` example of "Serde Data Model
Types". The composite vectors, a struct, an optional, an enum and a
sequence, are written byte by byte from the same specification's rules
and checked against `to_bytes`.

The suite also asserts the full 64-bit range in both directions, that a
struct written by the writer reads back with the `take_` functions, that
every refusal carries the offset it names, that UTF-8 is checked for
stray continuations, overlong forms, surrogates and values past
U+10FFFF, and that a writer passed on keeps its own bytes.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
