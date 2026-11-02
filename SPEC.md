# KOLLAPSE Specification

Status: Draft v0.1 — 2026-11-02
Format version: 1
File extension: `.klp`
License: Apache-2.0 (proposed, matching Krell)

KOLLAPSE is a lossless file encoder written in pure Krell. All codec logic
is Krell; host code provides only file I/O, allocation, and timing. The same
source builds for x86_64, AArch64, and RISC-V 64 through the Krell v1.1
toolchain, and encoder output is byte-identical on every target for the
same input, settings, and shared-state versions.

The process KOLLAPSE implements is **Regenerative Encoding**: wherever
possible, do not store the file — store the shortest description from
which the decoder regenerates it.

## 1. Limits and the accounting rule

These are properties of information, not implementation gaps, and the
format is designed around them:

1. An arbitrary file cannot be losslessly compressed to a few bytes.
   A payload of `n` bytes can name at most `2^(8n)` distinct files
   (plus shorter payloads), so almost all files of any size have no
   short description.
2. A few-byte payload is legitimate only when the missing information
   lives in shared state both sides already hold: a generated file
   (seed mode), a library entry (library mode), or a shared
   dictionary / model.
3. **Accounting rule:** every compression claim for KOLLAPSE states
   three numbers — payload size, header size, and the size of any
   shared state that is not a published, versioned part of the format
   itself. A ratio that does not count required shared state is not a
   KOLLAPSE ratio.

Incompressible data must never expand by more than the header plus
0.2%: any block that does not shrink is stored raw (codec 0).

## 2. Terminology

- **Payload** — the bytes in the `.klp` file after the header.
- **Shared state** — data the decoder must already hold, identified in
  the header by kind, ID, and size: a dictionary, a model, a generator
  version, or a library.
- **Generator** — a versioned, built-in deterministic function in the
  generator registry. Generators are never embedded in a `.klp` file
  in format v1.
- **Block** — an independently decodable chunk of the original file.
  Block independence gives streaming, parallel encode/decode, seeking,
  and error isolation.
- **Tier** — the encoder effort profile: FAST, BALANCED, or MAX.
  The tier is a property of encoding, not of the file: any conforming
  decoder decodes any tier.

## 3. Modes

| Mode | Value | Meaning |
|------|-------|---------|
| STORE | 0 | No compression. Framing, integrity, and metadata only. |
| FAST | 1 | LZ77 + FSE. Target: LZ4-class speed, better ratio. |
| BALANCED | 2 | Dictionary + larger window + adaptive range coding. |
| MAX | 3 | Context mixing. Slow encode, best ratio, decode kept as fast as the model allows. |
| SEED | 4 | File is the output of a registered generator. Payload is generator ID, seed, parameters, output length, and output hash. |
| LIBRARY | 5 | File is a library entry, or a library entry plus a delta payload that transforms it into the target. |
| DELTA | 6 | File is encoded as a delta against a base file identified by hash. |

The encoder tries the modes that apply and emits the smallest total
result under the accounting rule. SEED and LIBRARY are attempted only
when the corresponding shared state is present at encode time.

## 4. File format

All integers are little-endian. All offsets are from byte 0.

### 4.1 Header (fixed 64 bytes)

| Offset | Size | Field |
|--------|------|-------|
| 0 | 4 | Magic: `4B 4C 50 01` (`"KLP"` + version byte) |
| 4 | 2 | `format_version` = 1 |
| 6 | 2 | `flags` (see below) |
| 8 | 1 | `mode` (Section 3) |
| 9 | 1 | `shared_kind`: 0 none, 1 dictionary, 2 model, 3 generator, 4 library, 5 delta base |
| 10 | 2 | Reserved, must be 0 |
| 12 | 16 | `shared_id`: first 16 bytes of the SHA-256 of the shared state (all zero when `shared_kind` = 0) |
| 28 | 8 | `shared_size`: byte size of the shared state (accounting; the decoder verifies its local copy by ID, not size) |
| 36 | 8 | `original_size` |
| 44 | 4 | `block_size`: uncompressed bytes per block; 0 = whole file is one block (required for SEED and LIBRARY without delta) |
| 48 | 4 | `block_count` |
| 52 | 8 | Reserved, must be 0 |
| 60 | 4 | `header_crc`: CRC32C of bytes 0–59 |

Flags:

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | HAS_TRAILER_HASH | Trailer (4.3) is present. Required in v1. |
| 1 | MULTI_BLOCK | `block_count` > 1 |
| 2 | HAS_DELTA | Payload includes a delta section (LIBRARY with patch, or DELTA mode) |
| 3–15 | — | Reserved, must be 0 |

A decoder must reject a file with a bad magic, an unsupported
`format_version`, any reserved field or flag set, or a bad
`header_crc` — before allocating based on any header value.

### 4.2 Block record (repeated `block_count` times)

| Size | Field |
|------|-------|
| 4 | `original_len` (≤ `block_size`, except the final block) |
| 4 | `encoded_len` |
| 1 | `codec`: 0 store, 1 LZ/FSE, 2 dictionary LZ + range coding, 3 context-mixed, 4 delta |
| 4 | `block_crc`: CRC32C of the uncompressed block |
| `encoded_len` | Payload bytes |

A decoder must bounds-check every length against the remaining file
size before reading, and verify `block_crc` before emitting a block.

### 4.3 Trailer (32 bytes)

SHA-256 of the complete uncompressed file. The decoder verifies it
after the final block and reports failure without presenting the
output as complete (see Section 8).

### 4.4 Mode payloads

- **SEED:** before the block records, a seed section replaces them:
  `generator_id` (u32, registry index), `registry_version` (u32),
  `seed_len` (u8, ≤ 32), seed bytes, `param_len` (u16), parameter
  bytes, then the trailer. `block_count` = 0. Decoding runs the
  generator for exactly `original_size` bytes and verifies the
  trailer hash.
- **LIBRARY (exact):** `shared_kind` = 4, `shared_id` identifies the
  library, `shared_size` its size; the single block record is codec 0
  with `encoded_len` = 4: the entry index (u32). Decoding copies the
  entry and verifies the trailer hash.
- **LIBRARY (with delta) / DELTA:** the base is identified by
  `shared_id`; block records use codec 4 against the base in order.

## 5. Shared state

- **Dictionary (`.kld`):** raw bytes trained from a corpus. Its ID is
  the first 16 bytes of its SHA-256. Dictionaries are published,
  versioned artifacts; a `.klp` file never embeds one in v1.
- **Model:** the context-mixing model's parameter set for MAX mode,
  versioned with the format. A new model is a new `shared_id` and
  requires a format minor version.
- **Generator registry:** an append-only, versioned list. An entry,
  once published, never changes its output for a given seed and
  parameters. Removal is forbidden; deprecation is a note, not a
  deletion, because old `.klp` files must decode forever.
- **Library:** a versioned collection of files, indexed in the order
  they were added. Its ID covers the full ordered list of entry
  hashes, so any change is a new library ID.

## 6. Codec tiers (encoder requirements)

- **FAST:** sliding-window LZ77, 64 KiB window, hash-chain match
  search with a fixed small effort budget, literals and match headers
  entropy-coded with FSE (tANS). Single pass, streaming, no lookahead
  beyond the current block.
- **BALANCED:** 8 MiB window, optional dictionary pre-load, optimal
  parse within a bounded cost model, adaptive binary range coding.
  Optional reversible transforms detected per block: delta coding,
  RLE, and (for text-like blocks) BWT.
- **MAX:** context mixing — a set of context models whose predictions
  are mixed and coded with a binary range coder, in the PAQ family.
  Encode may be orders of magnitude slower than FAST. Decode uses the
  same model and must remain deterministic and target-identical.

Per block, the encoder may fall back to any cheaper codec, including
STORE, and records the choice in the block record.

## 7. Krell implementation

- Modules: `core` (framing, varints, CRC32C, SHA-256), `lz` (FAST),
  `dict` (BALANCED), `mix` (MAX), `regen` (SEED / LIBRARY / DELTA),
  `cli` (host interface).
- Host externs are limited to: read, write, allocate, free, and a
  monotonic clock for benchmarking. No codec decision may depend on a
  host value other than the input bytes and the settings.
- Determinism: same input + settings + shared-state IDs ⇒
  byte-identical `.klp` on x86_64, AArch64, RISC-V 64, and the LLVM
  lane. Any construct whose cost or result differs by target is out
  of the codec, following Krell's no-cliff rule.
- Streaming API: the encoder and decoder process one block at a time
  with caller-owned buffers; neither ever requires the whole file in
  memory.
- Resource limits: a decoder must enforce configurable caps on
  `original_size`, block size, and (in SEED mode) generator steps,
  and reject files that exceed them.

## 8. CLI

```
kollapse compress   <in> [-o <out.klp>] [--tier fast|balanced|max]
                    [--dict <file.kld>] [--seed] [--library <lib>]
kollapse decompress <in.klp> [-o <out>]
kollapse inspect    <in.klp>     # header, mode, blocks, accounting
kollapse bench      <in>         # all tiers: ratio + MB/s, both ways
kollapse dict build <corpus...> -o <file.kld>
```

Exit codes: 0 success; 1 usage; 2 input error; 3 format error
(corrupt or unsupported `.klp`); 4 integrity failure (CRC or trailer
hash mismatch); 5 required shared state not available.

On integrity failure during decompression, partial output is deleted
unless the user passed `--keep-partial`, and the exit code is 4.

`inspect` always prints the accounting triple from Section 1:
payload bytes, header bytes, shared-state bytes.

## 9. Conformance

A conforming implementation must pass:

1. **Round-trip:** every file in the test corpus compresses and
   decompresses byte-identically, in every applicable mode.
2. **Incompressibility:** random data never expands beyond header +
   0.2%.
3. **Decoder fuzzing:** no crash, no out-of-bounds access, and no
   unbounded allocation on mutated `.klp` inputs.
4. **Cross-target differential:** identical encoder output and
   identical decoder output on all Krell backends.
5. **Accounting:** `inspect` output matches the actual file and
   shared-state sizes.

Benchmarks are reported per tier as ratio and MB/s in both
directions, on named corpora, with shared-state sizes stated.

## 10. Security notes

A `.klp` decoder is a parser of untrusted input. All lengths are
checked before use, all allocations are capped (Section 7), and the
trailer hash is verified before success is reported. SEED mode runs
only registered, built-in generators under a step cap — v1 files
carry no executable content. Encryption is out of scope for v1: the
format is a compressor, and confidentiality belongs in a separate
layer applied before or after compression (note that encrypted data
is incompressible, so the order is compress first, then encrypt).

## 11. Roadmap

- **Phase 0:** this specification; format v1 frozen for review.
- **Phase 1:** `core` + FAST tier + CLI; conformance tests 1–3.
- **Phase 2:** BALANCED tier, dictionary build and format;
  cross-target differential (test 4).
- **Phase 3:** SEED and LIBRARY modes, generator registry v1.
- **Phase 4:** MAX tier (context mixing); DELTA mode.

## 12. Open questions

1. FSE table representation in the block payload: transmitted per
   block, or derived from a first pass over the block's statistics?
2. MAX model size: a small model that ships with the format, or a
   large published model counted as shared state?
3. Library governance: who can add entries, and how are libraries
   distributed?
4. Should `inspect` accounting amortize shared state over an assumed
   number of files, or always report its full size? (Draft position:
   always full size — no amortization in headline numbers.)
