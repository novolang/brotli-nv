# Changelog

All notable changes to brotli-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-01

The first implementation: every body is real, `stability = "experimental"`.
The decoder reads every stream in a corpus of 636 produced by libbrotli (nine
qualities, four window sizes, three modes, text, HTML, UTF-8, binary and 600 KB
sources) byte for byte, whole and fed one byte at a time into a one-byte
buffer. The encoder's output, 90 streams at five levels, is read by libbrotli.

### Added

- The decoder: stream header, meta-blocks (compressed, stored, metadata and
  last-empty), block types and block switching for all three categories,
  simple and complex prefix codes, context modes and maps with the inverse
  move-to-front, the distance cache and the direct and postfix codes, the 122784
  bytes of static dictionary and all 121 transforms, as data in
  `brotlidict.nv`. The transforms are checked against libbrotli's on seven
  words, among them one with a two-byte and one with a three-byte first
  character.
- The encoder, levels 0 to 4 (see the README for what each does and how it
  compares): hash chains, a lazy match from level 3, the static dictionary from
  level 2, Huffman codes limited to 15 bits, simple and complex code spelling
  with symbol 17 for runs of unused symbols, and the stored form when it is
  smaller.
- `brotlistream.BroReader` and `reader`, `peek_bits`, `read_bits`, `skip_bits`,
  `bits_left` and `align`: the bit reader, which sets a flag on a read past the
  end so a parser can rewind. This replaces the bitstream-nv dependency, which
  is removed (it would have needed `^0.0.1`, an interface release).
- `brotliprefix.BroTable`, `table_of`, `decode_with` and `read_code`;
  `brotlicontext.read_map_from`; `brotliblock.read_header_from`,
  `block_count_symbols`, `block_count_extra_bits`, `block_count_of`,
  `insert_code_for`, `copy_code_for`, `insert_range`, `copy_range` and
  `command_code`; `brotlidict.dictionary_data`, `index_bits` and
  `transformed_from`. The decoder needs the first group to read from one cursor
  and the encoder the rest; none was in the interface.
- `BroFault.BroBadDistance(distance, produced)`: a distance of zero or less, or
  past the output and not a dictionary word. `out_of_range_fault` answered
  `BroTruncated` in its documentation and had no honest variant to use.
- `BroPrefix.bits` and `BroContextMap.bits`, so `code_bits` and `map_bits` have
  something to answer.
- Suites: `brotli_vectors_tests.nv` (the reference streams, decoded whole, fed in
  pieces, cut short at every length and damaged byte by byte),
  `brotli_crafted_tests.nv` (streams built bit by bit for block switching in all
  three categories, metadata and stored blocks, and sixteen kinds of fault),
  `brotli_edge_tests.nv`, `brotli_encoder_tests.nv` and `tests/coverage.sh`.

### Changed

- **Breaking, signatures.** `brotlidec.step`, `brotlidec.feed`, `brotlienc.write`
  and `brotlienc.finish` take the decoder or encoder as a `var` argument, and
  `step`, `write` and `finish` take the buffer they write into as `var`: the
  compiler refuses a write through a parameter that is not `var` (E2036), and
  the pure shapes cannot carry the state a streaming decoder has to keep.
  `advanced` answers its argument unchanged. A decoder that has failed answers
  `BroDone` from `step`; `is_failed` and `fault_of` say why.
- **The distance cache is seeded newest first with 4, 11, 15, 16**, not 16, 15,
  11, 4. The interface and its suite had it the other way round; the reference
  decoder and RFC 7932 section 4 have the most recent distance as 4. Streams
  from libbrotli decode correctly only this way, and the assertions in
  `brotlidist_tests.nv` and `brotlicodec_tests.nv` were changed to match.
- `brotlistream.small_limits()` allows a 4 MiB window, not 64 KiB. The one-byte
  empty stream is a 22-bit window, and the suites decode it with these limits.
- `inverse_mtf([0, 1, 2])` is `[0, 1, 2]`, not a permutation of it; the
  assertion in `brotliblock_tests.nv` that it differs was replaced by one on
  `[1, 1, 0]`, which becomes `[1, 0, 0]`.
- `BroOutputFull` is the fault for a stream that expands past
  `BroLimits.max_output`.
- `brotlidec.window_bytes` answers the window the stream asked for, which is
  what the budget bounds; the ring behind it is the next power of two.
- A fault's position counts from the first byte ever fed, however the input
  was cut up.
- `brotlidict.transformed_len`, `max_transformed_len` and the other table
  readers answer 0 outside the table rather than panicking.
- The "RED ON PURPOSE" stamps are removed from the suites.

### Known

- `BroBadBlockType` and `BroBadTransform` are never produced by the decoder: a
  block-type code is read over `types + 2` symbols so it cannot name a missing
  type, and `resolve` checks the transform before a word is asked for.
- The decoder reads a meta-block's description (block types, context maps,
  prefix codes) whole, so fed one byte at a time it re-reads the part that has
  arrived at each byte. The cost is quadratic in the size of one description.
- A previous byte pair carries across meta-blocks for literal contexts, as the
  reference decoder does; the interface text said it was reset.
- The encoder uses one block type, one literal code and one distance code per
  meta-block of 128 KiB, and reaches back 128 KiB. It does not search the
  transforms, and searches words of eight bytes or more only.
- The large-window extension is not accepted.
- The decoder holds the whole window (a power of two at least the size the
  stream asked for) from the header on; a four-byte stream can cost the caller
  its budget, which is what the budget is for.

This release requires novo 0.23.0 or later.

## 0.0.2 — 2026-09-12

### Changed — The README is rewritten in plain technical-writer prose; no signature changed.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `brotlistream` — `BroLimits` as the caller's budget, the `(1 << bits)
  - 16` window, the large-window extension named separately, the
  `Content-Encoding` token, and the honest heuristic for a format with
  no magic number.
- `brotliblock` — the four meta-block shapes, the block-switch state
  with its seeded previous type, and insert-and-copy with the implicit
  distance flagged.
- `brotliprefix` — simple codes with their tie-break bit, complex
  codes, the Kraft sum as a number, and the single-symbol exception.
- `brotlicontext` — four context modes, the map with its inverse
  move-to-front, and the identity map for a block with one tree.
- `brotlidist` — `BroReference`, the seeded cache, the three ranges as
  three predicates, and `resolve`.
- `brotlidict` — the dictionary's size as a function, the five
  transform kinds, and `transformed_len` because it is not the word
  length.
- `brotlidec` — feed and drain, `BroNeedMore` and `BroWantsDrain` as
  two answers, `decompress` with no unbounded twin, and
  `has_checksum` answering false.
- `brotlienc` — quality 0 to 4, the gap to 11 as two numbers, and the
  three absent features as three predicates.
- `brotlierror` — thirteen refusals, and no `BroDistanceTooFar`.

### Known

- **`BroReference` is the load-bearing interface.** A distance past the
  output is a dictionary reference, not an error; a decoder that
  refused it rejects a large share of real HTTP responses and one that
  clamped it answers plausible garbage, which nothing catches because
  the format has no checksum.
- **The window is the caller's budget**, and so is the output size.
  A four-byte body can ask for a gigabyte.
- **The dictionary is 122,784 bytes and cannot be optional**, because a
  stream never says it used it.
- **The encoder's scope is assertable**, not a paragraph: the levels it
  implements, the level the format allows, and the two features it does
  not have are all functions.
- **Four silent-failure points are named** with a function each: the
  seeded distance cache, the `d0` push exception, the block type
  seeded to 1, and the implicit distance below code 128.
- **One name had to change.** `BroOutputFull` was a variant of both
  `BroFault` and `BroStep`, which is `E2031`; the step is
  `BroWantsDrain`.
- **`BroQuality` is `@value` and `BroEncoder` flattens it**, because
  SPEC § 14.5 keeps an unboxed struct out of a boxed struct's field.
  `quality_of` rebuilds the value.
- **No device claim.** A 122,784-byte dictionary settles it before the
  rest of the surface is considered.
- The scaffold's `src/brotli.nv` was dropped for nine prefixed
  modules.
