# Changelog

All notable changes to brotli-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
