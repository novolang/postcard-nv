# postcard-nv

Postcard is a compact binary format for typed values. A value is written
as its parts, one after another, with nothing between them: no field
names, no offsets, no framing. Both ends must already agree on the shape
of the data. The format is
[postcard](https://postcard.jamesmunns.com/wire-format), which is what
embedded Rust programs use for the payloads they send. This package
brings it to novo-lang.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

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
    match postcard.to_bytes(Reading { sensor: 3, celsius: 21.5 })
        Err(e) => println(e.message())
        // Nine bytes: 06 is the zigzag varint for 3, and the eight that
        // follow are 21.5 as a little-endian double.
        Ok(b)  => println(bytes.to_hex(b))
```

`postcard::from_bytes::<Reading>` in Rust reads the same nine bytes.

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: postcard.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `postcard` | The whole package: the two size questions, a `put_` and a `take_` function for every part of the format, the writer and reader that carry the standard library's serialisation traits, and the six named refusals. |

## How to choose an entry point

**`postcard.to_bytes` writes a novo-lang value through the standard
library's `Serialize` trait.** Reach for it when both ends are novo-lang
and the type has no optional member.

**The `put_` and `take_` functions write and read the format byte by
byte over a `Cursor`.** Reach for them when the other end is Rust and
its struct holds an unsigned integer or an `Option`, and for every read.
See rules 4 and 5.

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
   `i64` and `isize`. A Rust member of type `u32`, `u64` or `usize` is a
   plain LEB128 varint with no zigzag, and a `u8` or `i8` is one raw
   byte. Write those with `postcard.put_unsigned` and
   `postcard.put_byte`.
4. **A type with a `?T` member cannot be written through the trait.**
   `postcard.to_bytes` answers `PostcardOptionalMember`. The
   `Serializer` trait announces `None` and does not announce `Some`, so
   a writer has no hook in which to put the `0x01`, and `Some(5)` would
   go out as the same bytes as a plain `5`. Write the discriminant with
   `postcard.put_some` or `postcard.put_none` and the value after it.
   Reading an optional is fine: `postcard.take_option` and the trait's
   own reading hooks both work.
5. **Reading a whole value through the trait does not work yet.**
   `postcard.from_bytes` is published with nothing behind it.
   `Deserializer.field` answers a child cursor and leaves the parent
   where it was, which suits a format that carries names and cannot suit
   one that does not: member 2 begins where member 1 ended, and the
   parent never learns where that was. Every member would read the first
   value. Read with the `take_` functions over a `Cursor`, which does
   advance. One additive trait method, called after each member with the
   child that read it, would close this.
6. **An overlong varint is refused.** A varint whose continuation bits
   run past the width of the type is `PostcardOverlongVarint`. Accepting
   one would silently truncate the value.
7. **A discriminant that is neither 0 nor 1 is refused.** That covers a
   boolean byte and an optional's tag, and it is `PostcardBadTag`.
8. **A string's bytes must be UTF-8.** Otherwise `PostcardBadUtf8`.
9. **Reading one value out of a larger buffer leaves the rest.** The
   cursor's position is where the next value starts.
   `PostcardTrailingBytes` is for the caller that expected the buffer to
   hold exactly one value.
10. **Indexing into a sequence re-scans it.** Postcard writes no
    offsets, so element *i* is found by reading and discarding elements
    0 to *i* - 1. Walking a sequence by index therefore costs time
    proportional to the square of its length. Walk it in order instead.

## What is not included

- **A build for a microcontroller.** The surface speaks `Bytes`, `Str`
  and `Result`, none of which links on a device today, so this package
  makes no device claim and carries no probe. The device packages beside
  it are [rzcobs-nv](https://novo-lang.org/packages/rzcobs-nv),
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
- **32-bit floats.** postcard writes `f32` as four little-endian bytes,
  and novo-lang's `Float` is 64-bit.
- **Any input or output.** Every function here is arithmetic over bytes
  the caller already holds.

## Related packages

- [leb128-nv](https://novo-lang.org/packages/leb128-nv) and
  [zigzag-nv](https://novo-lang.org/packages/zigzag-nv) are this
  package's two dependencies. Postcard's integers are those two
  encodings, so they are declared rather than written again.
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
novo test tests/postcard_tests.nv      # 26 tests
```

Every vector is a payload that `postcard::from_bytes` reads and
`postcard::to_vec` would have produced, which is the only thing wire
compatibility can mean. The reference implementation is the `postcard`
crate and its wire specification.

The suite asserts that an unsigned varint is LEB128 and a signed one is
zigzag first, that a one-byte integer is not varinted, that a double is
eight little-endian bytes, that a string and a byte run open with a
varint length, that a sequence opens with its element count and an enum
with its variant index, that an optional is a discriminant and then the
value, that reading advances the cursor, that a buffer ending mid-value
is named as such, that an overlong varint and a bad discriminant are
refused, that a struct is its members end to end, that the round trip
through the traits holds for a type without an optional member, and that
a type with one is refused rather than written wrongly.

The tests compile today and fail at run, each on the `not implemented`
panic that is its body. That is the expected state of an interface
release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `postcard.max_varint_len`, `.unsigned_len`, `.signed_len` | no |
| `postcard.put_unsigned`, `.put_signed`, `.put_byte`, `.put_bool` | no |
| `postcard.put_f64`, `.put_str`, `.put_bytes` | no |
| `postcard.put_seq_len`, `.put_variant`, `.put_none`, `.put_some` | no |
| `postcard.take_unsigned`, `.take_signed`, `.take_byte`, `.take_bool` | no |
| `postcard.take_f64`, `.take_str`, `.take_bytes` | no |
| `postcard.take_seq_len`, `.take_variant`, `.take_option` | no |
| `postcard.writer`, `.writer_bytes`, `.reader`, `.reader_at` | no |
| `PostcardWriter`'s `Serializer` methods | no |
| `PostcardReader`'s `Deserializer` methods | no |
| `postcard.to_bytes`, `.from_bytes` | no |
| `postcard.PostcardError.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
