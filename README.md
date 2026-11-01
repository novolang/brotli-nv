# brotli-nv

Brotli is a lossless compression format, specified in
[RFC 7932](https://www.rfc-editor.org/rfc/rfc7932). It is the compression behind
`Content-Encoding: br`, which is most of what a browser downloads today. This
package reads and writes it in novo-lang, and performs no input or output
itself.

## What Brotli is

Brotli is an LZ77 compressor with prefix coding over it, like DEFLATE, plus
three things DEFLATE does not have.

**A static dictionary.** The format ships with 13504 built-in words of lengths 4
to 24, totalling 122784 bytes, and 121 *transforms* that modify a word —
capitalise it, add a prefix or suffix, omit characters from either end. A stream
can reference a transformed dictionary word instead of encoding it. That is how
a Brotli stream encodes `<!DOCTYPE html>` in a handful of bits on the first byte
of a response.

**Context modelling.** Which prefix code decodes the next literal depends on the
two bytes before it, through one of four context modes. The same applies to
distances, through the length of the copy.

**Block switching.** A meta-block is divided into runs, each with its own set of
prefix codes for literals, insert-and-copy commands and distances, so one stream
can carry text and binary without one spoiling the other's statistics.

A stream begins with a variable-length window field, then a sequence of
*meta-blocks*. A meta-block declares its uncompressed length, its prefix codes
and its context maps, and then a run of *insert-and-copy* commands: insert this
many literals, then copy this many bytes from this distance back.

| Value | Size |
| --- | --- |
| Static dictionary | 122784 bytes, 13504 words |
| Word lengths | 4 to 24 |
| Transforms | 121 |
| Window | `(1 << bits) - 16`, for bits 10 to 24 |
| Largest window under the large-window extension | 30 bits |
| Integrity check | none |

**Brotli has no checksum.** No CRC32, no XXH64, nothing.
`brotlidec.has_checksum()` says so as a value.

## Install

```
novo pkg add brotli-nv
```

## Example

Decompressing an HTTP response body, whose size the server chose and whose
ceiling you choose.

```novo ignore
use std.bytes
use brotlistream
use brotlidec

// `socket`, `read_some` and `flush` are the caller's own.
fn read_body(socket: Socket, var out: Bytes) -> Result<Int, BroFault> [io, net]
    var d = brotlidec.decoder(read_some(socket), brotlistream.small_limits())!
    var written = 0
    var going = true
    while going
        match brotlidec.step(var d, var out, written)
            BroProduced(n) =>
                written = written + n
            BroNeedMore =>
                brotlidec.feed(var d, read_some(socket))!
            BroWantsDrain =>
                written = written - flush(out, written)
            BroDone =>
                going = false
    match brotlidec.fault_of(d)
        Some(f) => Err(f)
        None => Ok(written)
```

Whole streams already in memory need none of that:

```novo
use std.bytes
use brotlidec
use brotlienc
use brotlistream

fn main() [io]
    match brotlienc.compress(bytes.from_str("hello hello hello hello"), brotlienc.best_declared())
        Err(e) => println(e.message())
        Ok(packed) =>
            match brotlidec.decompress(packed, brotlistream.small_limits())
                Ok(text) => println(bytes.to_str(text))
                Err(e) => println(e.message())
```

Build and test with:

```
novo pkg build
novo test
```

## What the package contains

| Module | Contents |
| --- | --- |
| `brotlistream` | The stream header: the window bits, the limits a caller allows, and the `Content-Encoding` token. |
| `brotliblock` | Meta-blocks, block switching, and the insert-and-copy command alphabet. |
| `brotliprefix` | Prefix codes in both their forms, simple and complex, and the completeness check. |
| `brotlicontext` | The four context modes, the context map, and the move-to-front decoding it uses. |
| `brotlidist` | The distance code space: the cache, the direct codes, the postfix codes, and `resolve`. |
| `brotlidict` | The static dictionary and its transforms. |
| `brotlidec` | The decoder: feed and drain over a window the caller sized. |
| `brotlienc` | The encoder, at quality 0 to 4. |
| `brotlierror` | `BroFault`, its message, and the two questions `is_budget` and `fault_at`. |

Nine modules, 141 public functions, 16 structs and 7 enums between them. No function opens a file, reads a clock
or waits for anything.

## How to choose an entry point

**You have a whole stream in memory: `brotlidec.decompress`.** It takes the
limits as an argument and has no unbounded form.

**The bytes arrive in pieces: `brotlidec.decoder`, `feed` and `step`.** See the
example above, and "Feeding and draining are two questions".

**You want to know what a stream asks for before decoding it:
`brotlistream.parse_header`.**

**You are producing a stream: `brotlienc`.** Read "What the encoder does" first.

## The distance that is not a distance

This is the one thing about Brotli that has no counterpart in another LZ format,
and it is where a decoder written from another one goes wrong.

```novo ignore
pub enum BroReference
    BroBackRef(distance: Int, length: Int)
    BroDictWord(length: Int, index: Int, transform: Int)
    BroDistOutOfRange(distance: Int, produced: Int)

pub fn resolve(p: BroDistParams, c: BroDistCache, code: Int, extra: Int,
               produced: Int, copy_len: Int) -> BroReference
```

In every other LZ format a distance is one thing: how far back in the output to
copy from, and a distance past the start of the output is a corrupt stream.

In Brotli a distance code means one of three things depending on where the
resolved number falls — the cache of recently used distances, a direct distance,
or the postfix-coded range. And then, whatever the code said, **a distance
larger than the number of bytes produced so far is a reference into the static
dictionary**. The excess selects a word by length and index, and the remainder
selects one of the 121 transforms to apply to it.

A decoder that gets this wrong does so in one of two ways. One refuses the
stream, and so rejects a large share of the HTTP responses on the web. The other
clamps the distance into the window and answers plausible rubbish that nothing
catches, because the format has no checksum.

`BroReference` makes neither mistake possible to write. A caller has to match on
it, and the dictionary arm is the one it cannot forget.

## The rules a user needs

1. **You decide the window size, not the stream.** A stream's window field can
   ask for up to 16 MiB, and up to a gigabyte under the large-window extension.
   A four-byte response body can ask for a gigabyte.
   `brotlistream.parse_header` answers what the stream asks for, `BroLimits` is
   what you allow, and `BroWindowTooLarge(want, limit)` carries both numbers when
   they disagree.
2. **`BroLimits.max_output` is the second, separate bound.** A window bound
   stops a large allocation; the output bound stops a small input expanding
   without end, which is the shape a decompression bomb takes.
3. **The window is `(1 << bits) - 16`, not `1 << bits`.** The minus sixteen is
   in the specification and is the field most readers of it miss. A decoder that
   used the round number accepts streams the format does not allow.
4. **The dictionary cannot be omitted.** A stream does not say whether it used
   one: it emits a distance past the end of its own output and expects the
   decoder to know. A decoder without the dictionary is not a smaller decoder;
   it is one that produces wrong bytes. The same goes for the transforms:
   `" of the "` is not a word in the dictionary, it is `"of"` with transform 7.
5. **The distance cache is seeded with 4, 11, 15, 16**, newest first, not with
   zeroes. A
   stream may reference the cache before it has produced a distance of its own,
   and a decoder that started from zeroes decodes the first symbols of *every*
   stream differently. `brotlidist.initial_cache`.
6. **Reusing the most recent distance does not move the cache.** Every other
   distance pushes onto it; the code that names the most recent one leaves it
   alone. `brotlidist.push` is where that exception lives, once.
7. **The previous block type starts at 1, not 0.** A stream whose first
   block-type code is the relative one selects type 1, and a decoder that seeded
   both to 0 decodes the whole block with the wrong prefix code.
   `brotliblock.initial_block_state`.
8. **An insert-and-copy code below 128 carries no distance**: it reuses the last
   one. A decoder that read a distance code for it consumes bits belonging to
   the next command, and everything after is noise.
   `brotliblock.uses_implicit_distance`.
9. **A metadata meta-block's bytes are not output.**
   `brotliblock.produces_output` answers the question.
10. **A prefix code with exactly one symbol is legal, and is not complete.**
    `brotliprefix.is_single_symbol` distinguishes it from a malformed one.
11. **There is no magic number.** The first bits *are* the window field, so no
    amount of looking at a buffer proves it is Brotli.
    `brotlistream.looks_like_brotli` is a heuristic and its name says so: it can
    rule out an obvious non-Brotli and can never confirm one. The real answer is
    the header.

Rules 5 to 10 are the ones that produce no error at all when broken, only
plausible bytes. Each has a named function and an assertion behind it.

The faults are `BroWindowTooLarge(want, limit)`, `BroBadWindowBits(bits)`,
`BroTruncated(at, need)`, `BroBadPrefixCode(at, total)`,
`BroBadSimpleCode(at, count)`, `BroBadMetaBlock(at, field)`,
`BroBadBlockType(at, kind, count)`, `BroBadContextMap(at, size)`,
`BroBadDictWord(length, index)`, `BroBadTransform(id)`,
`BroBadDistance(distance, produced)`, `BroLengthMismatch(declared, produced)`, `BroBadQuality(want)` and
`BroOutputFull(pending)`. `brotlierror.is_budget` distinguishes the ones that
are a limit meeting a well-formed stream from the ones that mean the stream is
wrong.

## Feeding and draining are two questions

`BroNeedMore` means the decoder has consumed everything it was fed and wants
more compressed bytes. `BroWantsDrain` means the caller's output buffer is full
and the decoder is holding finished bytes.

A single "not done" answer would leave a caller guessing which to do, and the
guess that feeds more into a full decoder is an unbounded buffer.

`brotlidec.feed` and `brotlidec.step` write into the decoder they are given,
so both take it as a `var` argument. The decoder keeps the input it has not
used and the window, and nothing else of the compressed stream: what it has
consumed is dropped at the next `feed`. `brotlidec.pending_input` is published
so a test can assert it. The decoder reads a stream in units and rewinds a
unit it could not finish; the unit is a command or a literal, except for a
meta-block's description (its block types, context maps and prefix codes),
which is read whole. So after a `BroNeedMore` the unread input is a few bytes,
or the part of one description that has arrived.

A decoder that has failed answers `BroDone`; `brotlidec.is_failed` and
`brotlidec.fault_of` say why, and `feed` refuses more input.

## The dictionary is 122784 bytes

The static dictionary is in the binary of every program that links the decoder.
A caller choosing Brotli over DEFLATE is choosing to carry 120 KB, and a number
in a README is not something a build can read, so `brotlidict.dictionary_bytes`
and `brotlidict.word_count` are functions and the test suite asserts them.

## What the encoder does

The reference implementation has eleven quality levels. This package declares 0
to 4, all of them greedy:

| Level | Candidates looked at | Lazy match | Dictionary |
| --- | --- | --- | --- |
| 0 | 1 | no | no |
| 1 | 4 | no | no |
| 2 | 16 | no | yes |
| 3 | 32 | yes | yes |
| 4 | 64 | yes | yes |

Input is cut into meta-blocks of 128 KiB. Each uses one block type, one literal
code and one distance code, and the encoder writes whichever of the compressed
and the stored form is smaller. The matcher reaches back across meta-blocks by
128 KiB, whatever window the stream declares. It uses the static dictionary for
words of eight bytes or more that appear unchanged; the 121 transforms are not
searched.

What lies above that — block splitting, context modelling and a search for the
cheapest parse — is most of the remaining compression ratio. An interface that
declared quality 11 and produced quality-4 output would be a claim nothing could
check, so the gap is a value instead. `brotlienc.quality_range()` is `[0, 4]`,
`brotlienc.format_max_quality()` is 11, `brotlienc.splits_blocks()` is false, and
`brotlienc.supports_custom_dictionary()` is false.

Measured against the reference compressor (libbrotli 1.1.0) at a 4 MiB window:

| Input | Bytes | Level 0 | Level 4 | Reference q5 | Reference q11 |
| --- | --- | --- | --- | --- | --- |
| Python source | 53688 | 17441 | 13829 | 13813 | 12197 |
| 600 KB of Python source | 600000 | 179297 | 133581 | 134073 | 128375 |
| An HTML page | 10220 | 367 | 268 | 263 | 205 |
| UTF-8 text in four scripts | 3360 | 131 | 105 | 104 | 103 |
| 20000 bytes of a short repeating pattern | 20000 | 881 | 584 | 508 | 437 |
| Random bytes | 5000 | 5004 | 5004 | 5004 | 5004 |

Level 4 is within a fraction of a per cent of the reference at quality 5 on
source code, and 10 to 15 per cent behind it on small repetitive inputs. Every
stream this encoder produces is a valid Brotli stream that any decoder reads;
the reference decoder read every one of the 90 streams the corpus above made at
the five levels.

The decoder runs at about 35 MB/s of output on text.

## What is not included

- **Quality 5 to 11.** See "What the encoder does".
- **Custom dictionaries.** RFC 7932's shared-dictionary extension, and the
  `Use-As-Dictionary` header the web is now shipping, are a different mechanism:
  the caller's own bytes prepended to the window rather than indexed by length.
  `brotlienc.supports_custom_dictionary` answers false.
- **The large-window extension.** `brotlistream.large_window_max_bits` names it
  as 30, but `parse_header` refuses its spelling of the window field with
  `BroBadWindowBits`, because a stream using it is not RFC 7932.
- **Threads.** Nothing in this package starts one.
- **An integrity check.** There is none in the format. A caller that needs to
  know the bytes are the ones that went in brings its own, and
  `brotlidec.has_checksum` is how it finds out without reading a specification.
- **A build for a microcontroller with no heap allocator.** A 122784-byte
  dictionary settles it: a device with kilobytes of RAM cannot link this decoder
  at all. A device that needs decompression has
  [lz4-nv](https://novo-lang.org/packages/lz4-nv), whose block decoder holds its
  state in a plain value and whose probe links for a Cortex-M4.
- **An HTTP dependency.** `Content-Encoding: br` is the consumer that makes this
  package worth writing, and a codec that pulled in an HTTP parser would make
  every program that decompresses a file download one.
  `brotlistream.content_encoding_token()` is `"br"`, published here where the
  format is, so a header writer can compare against it without either package
  depending on the other.

## Related packages

- This package reads bits through a reader of its own (`BroReader`, in
  `brotlistream`) and depends on nothing: Brotli is low-bit-first throughout,
  with no backward reads, and a read past the end sets a flag that lets a
  parser rewind instead of failing.
- [zstd-nv](https://novo-lang.org/packages/zstd-nv) answers the same window
  question the same way.
- [flate-nv](https://novo-lang.org/packages/flate-nv) is DEFLATE, gzip and zlib:
  no dictionary, no context modelling, and a checksum in both containers.

## A note on the layout

`BroQuality` is a plain value struct, because it is a parameter to five calls
and is built once per stream. The language does not admit such a struct as a
field of a boxed struct (SPEC section 14.5), so `BroEncoder` carries the four
numbers flat and `brotlienc.quality_of` rebuilds the value. A settings value
that is both passed and stored cannot be a plain value in both positions today.

One name had to change, and the compiler caught it. `BroOutputFull` was a
variant of both `BroFault` and `BroStep`, which is an error at the first call
site that constructs either: a variant is constructed by name, not by the type
the expression expects. The step is now `BroWantsDrain`.

## The reference implementation

[google/brotli](https://github.com/google/brotli). Its decoder is the
behavioural oracle and its `tests/testdata` corpus is the test data. RFC 7932 is
the specification, and the section numbers in the module headers are its.

Where the reference implementation and this package disagree about the bytes a
stream decodes to, the reference implementation is right.

## Licence

Apache-2.0. See `LICENSE`.
