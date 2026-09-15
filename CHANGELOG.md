# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-10

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Two surfaces.  `to_bytes` and `from_bytes` over the standard library's
  `Serialize` and `Deserialize`, so a novo-lang type writes itself; and
  `put_*` / `take_*` over a `Cursor`, so a caller can write the format
  byte for byte when the other end is Rust.
- `PostcardWriter` implements `Serializer` with every bracket method a
  no-op, which IS postcard: a struct is its members end to end, so
  `begin_struct`, `field` and `end_struct` have nothing to write.  The
  three that do write are `begin_seq` (the element count),
  `begin_variant` (the variant index) and the scalars.
- `PostcardReader` implements `Deserializer`, with `field` using its
  name for the error path only and `keys` answering the empty list —
  the wire carries no names.
- `PostcardError` naming the offset every fault was found at, because a
  format with no framing gives a reader nothing else to go on.
- `leb128-nv ^0.1.5` and `zigzag-nv ^0.1.4` as dependencies rather than
  as code: postcard's integers ARE LEB128 and ZigZag, and two copies of
  an encoding are two things to keep in step.

**Reading a novo-lang type back does not work yet**, and the release
says so rather than waiting.  `Serializer` is threaded — every method
takes the writer and returns it — and `Deserializer` is not:
`field(self, name)` answers a child cursor, the parent is unchanged, and
the walk asks the same parent for every member.  A self-describing
format scans for the key and needs no position; postcard has no names
and no offsets, so member 2 begins where member 1 ended and the parent
cannot learn where that was.  A cursor that advances reads a
three-member struct as `10 10 10`.  The impl is published with the
warning on it because the interface milestone exists to find this before
a body is written; until the standard library gains one additive hook,
read with the raw `take_*` surface, which advances because a `Cursor`
takes `mut self`.

**Three more places on the write side**, each with the raw surface as
its answer, all in the README where a reader will look.  novo-lang has
one integer type and postcard has ten integer encodings, so every `Int`
goes out as `varint(i64)` and a Rust `u32` member needs `put_unsigned`.
A `?T` member is **refused** — `to_bytes` answers
`Err(PostcardOptionalMember)` — because the `Serializer` trait announces
`None` and does not announce `Some`, so a writer cannot put the `Some`
discriminant in front of a present value.  And `seq_at(i)` re-scans,
because postcard writes no offsets.

**No device claim.**  There is no `tests/embedded_probe.nv`: the surface
speaks `Bytes`, `Str` and `Result`, and none of the three links at
`@tier(embedded)` today.  The device half of the deferred-logging story
is rzcobs-nv, bbqueue-nv and heapless-nv, which do carry probes.
