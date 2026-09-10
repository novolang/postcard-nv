# postcard-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The compact `serde` wire format.  A struct becomes its members'
encodings end to end with **nothing between them**: no names, no
lengths, no framing.  An `Int` is a zigzag varint, a `Float` is eight
little-endian bytes, a `Str` is a varint byte count and then the bytes,
a sequence is a varint count and then the elements, an enum is a varint
variant index and then the payload, and the unit is nothing at all.

A three-member struct of small integers is three bytes.  That is what it
is for.

## Adding it, and checking it

```bash
novo pkg add postcard-nv     # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test --isolate tests/postcard_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: postcard.<fn>`.  They turn green
one at a time as bodies land.

## The one example that will work

```novo
use std.bytes
use postcard

struct Reading
    sensor: Int
    celsius: Float

fn main() [io]
    match postcard.to_bytes(Reading { sensor: 3, celsius: 21.5 })
        Err(e) => println(e.message())
        Ok(b)  => println(bytes.to_hex(b))   // 060000000000803540
```

Nine bytes: `06` is zigzag(3), and the eight that follow are 21.5 as a
little-endian double.  `postcard::from_bytes::<Reading>` in Rust reads
the same nine.

## Where it sits in the deferred-logging story

novo-lang's deferred-logging design has two streams coming off a device,
and this package is the second one.

The **log** stream is `defmt`'s model: the device sends an interned
format-string index and raw argument bytes, and the host reconstructs
the text from the ELF.  It is self-describing *through the binary*,
which is why a log line costs a few bytes and no formatting code.
`rzcobs-nv` frames it, `leb128-nv` and `zigzag-nv` encode its integers,
`bbqueue-nv` buffers it.

The **payload** stream is this: typed values the device sends that are
not log lines — a configuration block, a sensor batch, a command reply.
Those need a schema the two ends agree on rather than a symbol table,
and postcard is what `postcard-rpc` and the embedded Rust ecosystem use
for exactly that.

Both streams can share the same framing and the same queue, and that is
the arrangement to expect: `cobs-nv` or `rzcobs-nv` around the frame,
this inside it.

**What `postcard-rpc` would add later**, and what this deliberately does
not: an endpoint and topic vocabulary, a request/response correlation
key, a varint sequence number, and the host half that dispatches on
them.  All of it sits *on top of* these bytes, so it is a package of its
own with this as a dependency rather than a section of this one.

## Two surfaces, and which one you are on

**The trait surface** — `to_bytes` and `from_bytes` over the standard
library's `Serialize` and `Deserialize` — writes a novo-lang type with
no code to write.  Reach for it when both ends are novo-lang.

**The raw surface** — `put_*` and `take_*` over a `Cursor` — writes the
format byte for byte.  Reach for it when the other end is Rust and its
struct has a `u32` in it, or an `Option` in it, because those are the
two things the trait surface cannot say — **and for every read**, until
the standard library's cursor is threaded (the section below).

## Where the trait surface falls short of the format

Three places on the write side, and one on the read side that is larger
than all of them and has its own section below.  Every one is a
consequence of the trait rather than of postcard, and every one has the
raw surface as its answer.

**novo-lang has one integer type and postcard has ten integer
encodings.**  `put_int` is the only integer hook the `Serializer` trait
offers, so every `Int` goes out as `varint(i64)` — zigzag then LEB128,
which is exactly what Rust postcard does for `i16`, `i32`, `i64` and
`isize`.  A Rust struct whose member is `u32`, `u64` or `usize` is
encoded as **plain LEB128 with no zigzag**, and a `u8` or `i8` as one
raw byte; none of the three round-trips through `put_int`.  So a
novo-lang type is wire-compatible with a Rust type whose integer members
are all `iN`, and needs `put_unsigned` / `put_byte` otherwise.

**A `?T` member is refused rather than written wrongly.**  postcard's
`Option` is a discriminant byte and then the value — `00` for `None`,
`01 <value>` for `Some`.  The `Serializer` trait announces `None` (as
`put_null`) and announces `Some` **not at all**: a present optional
member reaches the format as the bare value, with no hook in front of it
to write the `01`.  A writer that put the `00` in for `None` and nothing
in for `Some` would produce a document in which `Some(5)` and a plain
`5` are the same bytes, and a reader would take the second as a `None`
followed by garbage.

So `to_bytes` answers `Err(PostcardOptionalMember)` for any type with a
`?T` member.  A refusal is the only honest answer, and `put_none` /
`put_some` are how a caller writes the encoding by hand.  The **read**
side has no such problem — the trait's `opt_field` and `is_null` give a
reader exactly the hook it denies a writer — so a postcard document with
`Option`s in it can be read here even though it cannot be written.

**`seq_at(i)` re-scans.**  postcard writes no offsets, so element `i` is
found by decoding elements 0 through `i - 1` and discarding them.  A
walk that visits every element in order therefore costs O(n²).

## Reading a novo-lang type back does not work yet

The third item above is the small one.  This is the large one, and it is
worth its own section because it is the reason `from_bytes` is in the
surface with nothing behind it.

**`Serializer` is threaded and `Deserializer` is not.**  Every
`Serializer` method takes the writer and returns it, which is what lets
a format hold a write position — and it is why the write half of this
package is correct today.  `Deserializer.field(self, name)` answers a
**child** cursor and leaves the parent unchanged, and the read walk asks
the same parent for every member.

A self-describing format is fine with that: `field("beta")` scans the
document for the key `beta`, so the parent needs no position.  postcard
has no names and no offsets, so member 2 begins wherever member 1 ended
— and the parent has no way to learn where that was.  The position the
child advanced to is discarded with the child; the trait's methods take
`self` by value and declare no effects, so there is nowhere else to keep
it.

A cursor that carries a position and advances it — the only thing a
nameless format can do — reads a three-member struct like this:

```
expected: 10 20 30
actual:   10 10 10
```

Every member reads the first value, and nothing is refused.

**The impl is published anyway**, with this section as its warning,
because the interface milestone exists to find exactly this before
anyone writes a body.  What the standard library needs is one additive
method — a hook the walk calls after each member with the child that
read it, defaulted to the identity so no existing format changes:

```novo
fn after_field(self, child: Self) -> Self
    self
```

Until that lands, **read a postcard document with the raw surface**,
which works because a `Cursor` takes `mut self` and does advance:

```novo
var c = bytes.cursor_le(raw)
let sensor  = postcard.take_signed(c)!
let celsius = postcard.take_f64(c)!
```

Writing is unaffected: SPEC § 3.8.1 guarantees the write walk visits
members in declaration order, which is exactly what postcard needs, and
`to_bytes` is correct for every type without a `?T` in it.

## The layer, and why

`core`.  Everything here is arithmetic over bytes the caller already
holds, and no function declares an effect — a wire format has nowhere to
put one.

It carries **no `tests/embedded_probe.nv`**, so it makes no device
claim, and the audit's `core-embedded` row passes by saying so.  That is
deliberate: the surface speaks `Bytes`, `Str` and `Result`, and none of
the three links at `@tier(embedded)` today.  The device half of the
deferred-logging story is `rzcobs-nv`, `bbqueue-nv` and `heapless-nv`,
all three of which do carry a probe; this is the payload beside it, on
the host.

## The dependencies, and why they are dependencies

```toml
leb128-nv = "^0.1.5"
zigzag-nv = "^0.1.4"
```

postcard's integers **are** LEB128 and ZigZag.  Writing them again here
would be two copies of an encoding to keep in step, and the second copy
is the one that gets a bug fixed in the first.  Both are `core`, which
is what a `core` package may depend on.

## The reference implementation

`postcard` (MIT/Apache-2.0, James Munns) and its wire specification.
Every vector in `tests/postcard_tests.nv` is a payload
`postcard::from_bytes` reads and `postcard::to_vec` would have produced,
which is the only thing "wire-compatible" can mean.

## Status

| function | implemented |
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
