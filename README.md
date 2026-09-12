# brotli-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

RFC 7932 Brotli, decoder first. The compression behind
`Content-Encoding: br`, which is most of what a browser downloads.

- `brotlistream` — the header, the window bits, and the caller's budget;
- `brotliblock` — meta-blocks, block switching, insert-and-copy;
- `brotliprefix` — prefix codes, simple and complex;
- `brotlicontext` — context modelling and the context map;
- `brotlidist` — the distance code space; **the load-bearing module**;
- `brotlidict` — the static dictionary and its 121 transforms;
- `brotlidec` — feed and drain over a window the caller sized;
- `brotlienc` — the encoder, quality 0 to 4, scoped honestly;
- `brotlierror` — thirteen refusals, and the one that is not there.

```
novo pkg add brotli-nv
novo pkg build
novo test
```

## The one example that will work

Decompressing an HTTP response body whose size the server chose and
whose ceiling you choose.

```novo ignore
use brotlistream
use brotlidec

fn read_body(socket: Socket, out: Bytes) -> Result<Int, BroFault> [io, net]
    var head = read_some(socket)
    var d = brotlidec.decoder(head, brotlistream.small_limits())!
    var written = 0
    loop
        match brotlidec.step(d, out, written)
            BroProduced(n) =>
                written = written + n
                d = brotlidec.advanced(d)
            BroNeedMore =>
                d = brotlidec.feed(d, read_some(socket))!
            BroWantsDrain =>
                written = written - flush(out, written)
            BroDone =>
                break
    Ok(written)
```

## The load-bearing interface: `BroReference`

```novo ignore
pub enum BroReference
    BroBackRef(distance: Int, length: Int)
    BroDictWord(length: Int, index: Int, transform: Int)
    BroDistOutOfRange(distance: Int, produced: Int)

pub fn resolve(p: BroDistParams, c: BroDistCache, code: Int, extra: Int,
               produced: Int, copy_len: Int) -> BroReference
```

**In every other LZ format a distance is one thing**: how far back in
the output to copy from, and a distance past the start of the output is
a corrupt stream. In Brotli a distance code means one of three things
depending on where the resolved number falls — the last-distances
cache, a direct distance, or the postfix-coded range — and then,
whatever the code said, **a distance larger than the number of bytes
produced so far is a reference into the static dictionary.**

Not an error. The excess selects a word by length and index, and the
remainder selects one of 121 transforms to apply to it. That is how a
Brotli stream encodes `<!DOCTYPE html>` in a handful of bits on the
first byte of a response.

A decoder that gets this wrong gets it wrong in one of two ways, and
both are worth naming. One **refuses** the stream — and rejects a large
share of the HTTP responses on the web. The other **clamps** the
distance into the window and answers plausible garbage that nothing
catches, because **Brotli has no checksum**: no CRC32, no XXH64,
nothing. `brotlidec.has_checksum()` says so as a value.

`BroReference` makes neither mistake spellable. A caller has to
`match`, and the dictionary arm is the one it cannot forget to write.

## Three decisions worth arguing

### The window is the caller's budget

A Brotli stream opens with a variable-length `WBITS` field asking for a
sliding window of up to 16 MiB — and up to a gigabyte under the
large-window extension. A decoder that read the number and allocated
would be a denial of service with a decompressor on the front: a
four-byte response body can ask for a gigabyte.

So `brotlistream.parse_header` answers what the stream **asks for**,
`BroLimits` is what the caller **allows**, and `BroWindowTooLarge`
carries both numbers when they disagree. zstd-nv answers the same
question the same way and for the same reason; naming the shared shape
is more useful than two packages each discovering it.

`BroLimits.max_output` is the other half, and it is the one a naive
limit misses: a window bound stops a large *allocation*, and the output
bound stops a small input expanding without end — which is the shape a
decompression bomb takes. `brotlidec.decompress` takes the same limits
and has **no unbounded twin**.

Also: the window is `(1 << bits) - 16`, not `1 << bits`. The minus
sixteen is in the specification and is the field most readers of it
miss; a decoder that used the round number accepts streams the format
does not allow.

### The dictionary is 122,784 bytes, and it is not optional

The static dictionary holds 13,504 words of lengths 4 to 24 and is in
the binary of every program that links the decoder. A caller choosing
this over plain deflate is choosing to carry 120 KB, and a number in a
README is not something a build can read — so `dictionary_bytes()` and
`word_count()` are functions and the test suite asserts them.

It cannot be made optional, because a Brotli stream does not *say*
whether it used it. It emits a distance past the end of its own output
and expects the decoder to know. A dictionary-less decoder is not a
smaller decoder; it is one that produces wrong bytes.

And the 121 transforms are the other half, not a detail. `" of the "`
is not a word in the dictionary — it is `"of"` with transform 7. A
decoder with the words and no transforms is as wrong as one with
neither.

### The encoder's scope is a value, not a paragraph

The reference implementation has eleven quality levels. This package
declares **0 to 4**: greedy matching, a hash chain, a lazy match at 3
and 4, and prefix codes from the block's own histogram. What is above
that — block splitting and context clustering — is a research project,
and it is most of the remaining ratio.

An interface that declared `quality: 11` and produced quality-4 output
would be a lie the compiler cannot catch. So the gap is assertable:
`brotlienc.quality_range()` is `[0, 4]`,
`brotlienc.format_max_quality()` is 11, `brotlienc.splits_blocks()` is
false, and `brotlienc.supports_custom_dictionary()` is false. A caller
comparing output sizes against another library can name which feature
is missing rather than wondering.

Expect roughly 3 to 8 % larger output than the reference compressor at
quality 11 on HTML, and closer than that on text where the dictionary
does the work. Every stream it produces is a valid Brotli stream that
any decoder reads.

## The four places a decoder goes silently wrong

Each has a named function and an assertion, because none of them
produces an error — they all produce plausible bytes.

| | |
| --- | --- |
| **The distance cache is seeded** | 16, 15, 11, 4 — not zeroes. A stream may reference the cache before it has produced a distance of its own, and a decoder that started from zeroes decodes the first symbols of *every* stream differently. `brotlidist.initial_cache`. |
| **Reusing `d0` does not move the cache** | Every other distance pushes onto it; the code that names `d0` leaves it alone. `brotlidist.push` is where that exception lives, once. |
| **The previous block type starts at 1** | Not 0. A stream whose first block-type code is the relative one selects type 1, and a decoder that seeded both to 0 decodes the whole block with the wrong prefix code. `brotliblock.initial_block_state`. |
| **An insert-and-copy code below 128 carries no distance** | It reuses the last one. A decoder that read a distance code for it consumes bits belonging to the next command, and everything after is noise. `brotliblock.uses_implicit_distance`. |

Two more that are structural rather than silent: a metadata meta-block's
bytes are **not** output (`brotliblock.produces_output`), and a prefix
code with exactly one symbol is legal and is **not** complete
(`brotliprefix.is_single_symbol`).

## Feeding and draining are two different questions

`BroNeedMore` means the decoder has consumed everything it was fed and
wants more compressed bytes. `BroWantsDrain` means the caller's output
buffer is full and the decoder is holding finished bytes. A single "not
done" answer would leave the caller guessing which to do, and the guess
that feeds more into a full decoder is an unbounded buffer.

`brotlidec.pending_input` is published so a test can assert the decoder
is not accumulating: it is bounded by the longest single symbol the
format can write, which is a constant and not a function of the input.

## `Content-Encoding: br`, without a dependency

http-codec-nv names `Content-Encoding: br` and this package decodes it,
and a dependency in either direction would be wrong — a codec that
pulled in an HTTP parser, or an HTTP parser that pulled in every codec.
So the spelling is published here, where the format is:
`brotlistream.content_encoding_token()` is `"br"`, and a header writer
compares against it.

Brotli has **no magic number** — the first bits *are* the window field —
so there is no way to look at a buffer and know it is Brotli.
`brotlistream.looks_like_brotli` is a heuristic and the name says so; it
can rule out an obvious non-Brotli and can never confirm. The real
answer is the header.

## What this does not do, by name

| | |
| --- | --- |
| **Quality 5 to 11** | Block splitting and context clustering. A row of its own; `format_max_quality` publishes the gap. |
| **Custom dictionaries** | RFC 7932's shared-dictionary extension and the `Use-As-Dictionary` header the web is now shipping are a different value — the caller's own bytes prepended to the window rather than indexed by length. `supports_custom_dictionary` says false. |
| **The large-window extension** | `large_window_max_bits` names it as 30 and `parse_header` reads the flag, but a stream using it is not RFC 7932 and accepting one is a decision a caller makes with its own budget. |
| **Threads** | Single-threaded by construction; a `core` package has no way to start one. |
| **Integrity** | There is none in the format. A caller that needs to know the bytes are the ones that went in brings its own, and `has_checksum` is how it finds out without reading a paragraph. |

## Dependencies

| | |
| --- | --- |
| `bitstream-nv ^0.0.1` | The LSB-first bit reader. The same dependency zstd-nv takes for the same reason: a bit accumulator is four integers and a shift, every codec on the grid needs one, and three of them would be three places for the same off-by-one to live. Brotli is LSB-first throughout with no backward reads, so unlike zstd-nv this package needs nothing the published surface does not have. |

Nothing else, and **not http-codec-nv** — see above.

## Where the language decided the layout

`BroQuality` is `@value` because it is a parameter to five calls and is
built once per stream. SPEC § 14.5 keeps an unboxed struct out of a
boxed struct's field, so `BroEncoder` carries the four numbers flat and
`brotlienc.quality_of` rebuilds the value. It is a small tax and it is
recorded rather than hidden: a settings value that is both *passed* and
*stored* cannot be `@value` in both positions today.

One name had to change, and the compiler caught it: `BroOutputFull` was
a variant of `BroFault` **and** of `BroStep`, which is `E2031` at the
first call site that constructs either — a variant is constructed by
name, not by the type the expression expects. The step is now
`BroWantsDrain`, which also reads better.

## The layer, and why

`core`. A compressed stream is bytes somebody else read — an HTTP body,
a file, a cache entry — and turning it back into bytes is arithmetic
over a window the caller sized. The decoder is a feed-and-drain state
machine, which is how a `core` package takes a stream it must not
accumulate.

The window is the one number that could have made this `host`, and it
is answered by making the budget an argument rather than by allocating
on a number out of a file.

**No `@tier(embedded)` claim.** A 122,784-byte dictionary settles it: a
device with kilobytes of RAM cannot link this decoder at all, whatever
the rest of the surface looks like. `docs/publishing.md` says a device
claim is built and not asserted, and there is nothing here to build one
from. A device that needs decompression on the grid has lz4-nv, whose
block decoder is `@value` state and whose probe links for a Cortex-M4.

## The reference implementation

`google/brotli` (MIT) — the reference decoder is the behavioural oracle
and its `tests/testdata` corpus is the test data, as the plan's row for
this package says. RFC 7932 is the specification, and the section
numbers in the module headers are its.

Where the reference implementation and this package disagree about the
bytes a stream decodes to, the reference implementation is right.

## Status

| | |
| --- | --- |
| version | 0.0.1, `stability = "draft"` |
| modules | 9 |
| public functions | 119, every body a `todo()` |
| public types | 8 boxed structs, 6 `@value` structs, 7 enums with 36 variants |
| tests | 32, 232 assertions, red until bodies land |
| device claim | none, and § The layer says why |
