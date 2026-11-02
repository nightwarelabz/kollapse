# KOLLAPSE — Spec Analysis and Compression Research

Date: 2026-11-02
Scope: a design review of SPEC.md draft v0.1 against the current
lossless-compression state of the art. Part 1 is analysis (findings and
recommendations for the spec). Part 2 is the research digest with
sources. Part 3 verifies the two benchmark claims the design was
founded on. Numbers are as reported by their sources; items that
could not be independently confirmed are flagged as such.

## Part 1 — Analysis of SPEC v0.1

The spec's foundations are sound: the counting bound is stated
correctly, the accounting rule (payload + header + required shared
state) matches the Hutter Prize's total-size rule, block independence
buys streaming/seeking/error isolation, and the mode set covers the
space honestly. The findings below are refinements, ordered by how
much they should change before Phase 1 freezes the format.

### A1. Resolve §12.1 (FSE tables): transmit normalized counts

Zstd's design answers this directly. Its sequence codes are
block-static FSE distributions whose normalized counts are transmitted
in a compact header form; literals use Huffman with its own per-block
mode (predefined / RLE / transmitted / repeat). Deriving the table
from a first pass over the block does not remove the need to transmit
the distribution — it only changes who computes it. **Recommendation:**
specify a zstd-style normalized-count table description for codec 1,
with table-log capped by block size. This also keeps the decoder
single-pass.

### A2. Resolve §12.2 (MAX model): small built-in model, versioned with the format

Every competitive context-mixing system (PAQ family, cmix) ships its
model as code plus constants — part of the program, not an external
file — and the Hutter accounting counts the decompressor, model
included. A small built-in model versioned with the format keeps MAX
files self-contained and keeps the accounting simple: nothing to
count beyond the payload. **Recommendation:** v1 MAX ships a compact
built-in model; large external models, if ever offered, are shared
state under the existing rule.

### A3. DELTA mode: define it as LZ over the base, not a separate instruction stream

The spec leaves codec 4's representation open ("delta against the
base in order"). The field offers two shapes: explicit copy/insert
instruction streams (VCDIFF/xdelta3, bsdiff) and dictionary-style
patching (zstd `--patch-from`, which is literally "dictionary
compression with windowSize > srcSize"). Measured results favor the
second for a design like this one: on git tarballs, zstd patch-from
at level 19 produces patches comparable to bsdiff's, while running
100–200× faster at low levels; a secondary compilation of patch sizes
(as % of new file) puts zstd patch-from at 7.57% vs bsdiff 8.17% and
xdelta3 13.60% (secondary source — treat as indicative). The decisive
argument is architectural: base-as-window reuses the codec 2 pipeline
(LZ + range coding, literals fully compressed), so DELTA and LIBRARY
-with-delta become one code path — `shared_kind = 5`, codec 2/4
against a preloaded window. Explicit copy/insert wins only where
random access into the patch or near-zero decode memory matters, and
bsdiff's approximate-matching trick (a diff stream for shifted data)
is the one capability the window approach lacks. **Recommendation:**
define codec 4 as LZ with the base preloaded as window, in v1;
consider a bsdiff-style codec later only if binary patching becomes a
stated goal.

### A4. Name the block-independence tradeoff in the spec

Independent blocks reset the LZ window and all adaptive context at
every block boundary. Zstd, by contrast, keeps its window across the
blocks of a frame. The ratio cost is small for large blocks and real
for small ones — most acute for MAX, where the mixer must re-warm per
block. The benefits (parallelism, seeking, error isolation) justify
the choice, but the spec should state the cost and give per-tier
default block sizes (large for BALANCED/MAX, smaller for FAST
streaming). Worth one Phase 2 experiment: chained blocks that inherit
window/context vs fully independent blocks, measured on Silesia and
enwik8.

### A5. The expansion bound needs exact wording

§1 promises incompressible data expands by no more than "the header
plus 0.2%", and conformance test 2 repeats it. The actual overhead is
64 (header) + 32 (trailer) + 13 × block_count (block records) +
payload growth. For small files that exceeds "header + 0.2%" as
written. **Recommendation:** define the bound exactly —
`total ≤ original + 96 + 13 × block_count + 0.2% × original` — and
make test 2 check that formula.

### A6. Pin integer-only adaptation for determinism

§7 requires byte-identical output on every target. Adaptive coders
(BALANCED's range coder, MAX's mixer) drift if any probability update
uses floating point, because float rounding is a portability hazard
even under Krell's total semantics. PAQ-family mixers are already
fixed-point designs (stretch/squash tables), so this costs nothing.
**Recommendation:** add one sentence to §6/§7: all adaptive
probability updates use integer arithmetic only.

### A7. `dict build`: name the algorithm class and size norms

Practice (zstd) trains dictionaries with the COVER algorithm
(d-mer frequency optimization, d = 6–8), from >100 samples totaling
≈100× the dictionary size; the customary cap is ~110 KiB (zstd's
`--maxdict` default is 112,640 bytes). Gains concentrate on small
files: zstd's documentation reports ~10% at 64 KB files up to ~5×
below 1 KB. Meta's published case study: ~1,000 GitHub-user JSON
records (~850 KB) compressed to >300 KB naively (2.8×) and to 122 KB
with a 65,599-byte trained dictionary (6.9×; honest total counting
the dictionary once ≈ 187.6 KB). **Recommendation:** §5/Phase 2
should name COVER-style training as the reference algorithm and adopt
the ~110 KiB default cap, so `.kld` files from different
implementations stay comparable. Note also RFC 9842 (Compression
Dictionary Transport, September 2025): the web platform has now
standardized shared-dictionary compression (Brotli/Zstd dictionary
formats negotiated by hash) — the dictionary/library direction is
industry-validated, and the `.kld` ID scheme (SHA-256 prefix) matches
its Available-Dictionary hash approach.

### A8. FAST tier: calibrate the target to zstd, not raw LZ4

"FSE-coded literals and match headers at LZ4-class speed" is really
zstd's design point, not LZ4's — LZ4 does no entropy coding at all.
Reference numbers (Silesia, single core, zstd project's lzbench
table): LZ4 2.101× at 675 MB/s compress / 3,850 MB/s decompress;
zstd -1 2.896× at 510 / 1,550 MB/s. **Recommendation:** state the FAST
target as zstd-1-class (≈2.9× at hundreds of MB/s both ways) rather
than LZ4-class speed; it is the honest bar for an LZ+FSE design and
still decisively faster than the BALANCED tier.

### A9. Name the benchmark corpora in §9

"Named corpora" should be named: enwik8 (development-scale text),
enwik9 (the Hutter/LTCB frontier), Silesia (mixed-type speed
benchmarks), Calgary/Canterbury (small-file behavior), plus two
regenerative corpora the classical set lacks: a generated-files
corpus for SEED demonstrations, and a similar-files corpus (e.g.
consecutive releases of a source tree, mirroring the git-tarball
patching evaluations) for LIBRARY and DELTA. Without the last two,
the modes that make KOLLAPSE distinctive have no standard yardstick.

### A10. MAX tier: set speed expectations from the record table

Context mixing at the frontier is not "slow" in the usual sense — it
is hours per gigabyte in both directions. From the Large Text
Compression Benchmark (LTCB): the #1 entry runs 220,071 ns/B
(≈61 hours/GB) to compress enwik9 and ≈67 hours/GB to decompress;
paq8px at its largest setting runs 291,916 ns/B and uses ~28 GB RAM.
KOLLAPSE's MAX will live orders of magnitude below these ratios with
a small model, but the tier's decode cost should be framed honestly
in the spec: MAX is for ratio-critical data of bounded size, not a
general-purpose setting.

### A11. SEED and LIBRARY have strong precedents — cite them

- **.kkrieger** (2004): a complete FPS in 97,280 bytes, versus the
  developers' 200–300 MB conventional estimate — a ~2,000–3,000×
  regenerative ratio. Textures are stored as creation history, meshes
  as deformed primitives; everything is generated at load (~300 MB
  RAM in use). It is also the canonical accounting caveat: the OS and
  DirectX it stands on are not counted — exactly the shared state
  KOLLAPSE's rule forces into the open.
- **Seed worlds:** Elite generated 8 galaxies × 256 planets from
  seeds; No Man's Sky derives 2^64 planets from one 64-bit seed.
  SEED mode is this practice, formalized with hashes and a registry.
- **Library learning (DreamCoder/Stitch):** the MDL objective —
  bits(library) + bits(data | library) — is the formal version of the
  accounting rule applied to LIBRARY mode: a library entry earns its
  place only if total description length falls.
- **Kolmogorov framing:** K(x) is uncomputable, and the identity
  K(x) ≤ compressed size + decompressor size is precisely the
  accounting rule. The spec is, in effect, Kolmogorov complexity with
  honest bookkeeping and computable approximations.

### A12. Neural compression: only the small-counted-model direction survives the accounting rule

LLM-based arithmetic coding posts spectacular ratios with the model
excluded — Chinchilla 70B scores 0.664 bits/byte on enwik9, but its
~140 GB of weights, counted, would be ~14,008% of the 1 GB input.
Bellard's ts_zip (RWKV 169M) reaches 135,443,237 bytes on enwik9
before counting its ~169 MB model; counted, a single file totals
~304 MB — worse than xz. Speeds are 0.2–5 kB/s class, 100–1000×
slower than classical coders. The one neural direction compatible
with the accounting rule is the one that just moved the Hutter
record: small pretrained transformers *inside* cmix derivatives,
counted in the total (one entry added a 6M-parameter transformer for
+2.9 MB of counted size). **Recommendation:** no neural tier in v1;
if MAX ever grows a learned component, it must be small, counted,
and versioned like any other shared state.

## Part 2 — Research digest

### 2.1 The frontier (enwik9, 1 GB of Wikipedia text)

The Large Text Compression Benchmark (LTCB, Matt Mahoney,
mattmahoney.net/dc/text.html, updated 2026-10-14) ranks by total
size = compressed file + decompressor, the same accounting KOLLAPSE
uses:

| Entry | Total bytes | Compress | RAM |
|---|---|---|---|
| ex-girlfriend 1.0 (LTCB #1) | 91,778,237 | 220,071 ns/B ≈ 61.1 h/GB | ~10 GB |
| cmix-neif | 91,913,343 | 175,392 ns/B | 9,750 MB |
| zmix | 96,096,261 | — | — |
| nncp v3.2 | 107,261,318 | 241,871 ns/B | — |
| cmix v21 (vanilla) | 108,244,767 | 622,949 ns/B ≈ 173 h/GB | 30,950 MB |
| paq8px v206fix1 -12L | 125,099,359 | 291,916 ns/B | 28,151 MB |
| xz 5.2.1 (LZMA2, 1 GiB dict) | 197,368,568 | 5,876 ns/B | — |

(LTCB's zstd and brotli enwik9 entries are from 2016-era versions —
zstd 0.6.0 at 215,674,670 and brotli at 223,597,884 — and understate
current releases; treat them as historical.)

The Hutter Prize (awarded track: total ≤ 99% of the previous winner,
≲50 h, single core, <10 GB RAM) moved in 2026 from 110,793,128 bytes
(September 2024) to 100,424,672 (fx2-cmix-T, 2026-07-24) — the
largest single-year advance in the prize's 20 years, driven by small
pretrained transformers inside cmix derivatives, counted in the
total. A later self-reported result, cmix-views 1.0, claims
90,515,901 bytes including decoder, dictionary, and model at 58 h
18 m encode (GitHub, 2026-10-28) — not yet in the LTCB table;
self-reported, so treat as provisional.

On the standard small corpora, paq8px v217 (results September 2026)
leads Calgary (568,427), Canterbury (300,299), and Silesia at its
largest setting (27,682,006), ahead of cmix v21 on each.

### 2.2 The speed tier (Silesia, single core, lzbench via the zstd project)

| Codec | Ratio | Compress | Decompress |
|---|---|---|---|
| lz4 1.10.0 | 2.101 | 675 MB/s | 3,850 MB/s |
| zstd 1.5.7 -1 | 2.896 | 510 MB/s | 1,550 MB/s |
| brotli 1.1.0 -1 | 2.883 | 290 MB/s | 425 MB/s |

These are one machine's numbers (Core i7-9700K) and move with
hardware, but the ordering is stable: entropy coding costs speed and
buys roughly 0.8 ratio points at level 1.

### 2.3 Dictionaries

Covered analytically in A7. Additional data points: zstd pull-request
measurements show HTML-header corpora improving from 2.32→3.41 (0–8
KB files) and 3.03→4.92 (8–16 KB files) with a 110 KB dictionary.
Brotli's built-in static dictionary is 122,784 bytes (13,504 words
and phrases, 121 transforms) and buys ~7% on ~50 kB documents, with
no density advantage at ≥1 MB — a built-in dictionary is a small-file
feature. RFC 9842 (2025) standardizes dictionary transport for the
web (Chrome/Edge 130+); Google reports Search HTML at −23% versus
plain Brotli (secondary summary). A 2026 preprint (DictForge) claims
further gains over COVER (+3.1–14.3% at zstd L3) — preprint,
unverified.

### 2.4 Delta encoding

Covered analytically in A3. Format facts: VCDIFF (RFC 3284) uses
ADD/COPY/RUN instructions over windowed source data with separate
data/instruction/address streams and a 256-entry code table;
xdelta3 is a VCDIFF implementation with pluggable secondary
compressors. Bsdiff uses suffix-array approximate matching with
control/diff/extra streams; Percival's measurements on 97 FreeBSD
binaries: 36,397,575 bytes of new files → bsdiff patches 621,277
(58.3×) vs Xdelta 3,288,540 (11.0×); bsdiff's cost is memory
(17× the old file). Zstd `--patch-from` documents itself as
dictionary compression with the window covering the source; the
1.5.7 release made levels 18/19 patching 4.4–4.9× faster than 1.5.6.

### 2.5 Entropy coders

The coders are within bytes of each other on ratio (ryg_rans tests:
rANS output 435,113–435,120 bytes vs 435,378 for 12-bit arithmetic
coding on the same file) — ANS is a speed technology, not a ratio
one. Reference speeds (dated hardware, ordering still instructive):
tANS/FSE ≈350 MB/s encode / ≈500 MB/s decode; range coding ≈100–150
/ ≈80; binary arithmetic ≈20–30 MB/s. Production rANS uses
interleaved streams, and the interleave count is a format-level
constant with a machine-dependent sweet spot (Giesen, 2026) — a
Krell no-cliff consideration for codec 1's FSE design: fix the
interleave in the format, do not tune it per target. Zstd's
block-static approach (Huffman literals ≤11 bits in 4 parallel
streams; FSE sequence codes; per-block mode selection) is the proven
template for FAST. Huffman's worst case is ~1 bit/symbol overhead;
arithmetic coding's is ~2 bits per message — the reason block-static
FSE wins at speed and adaptive arithmetic wins at ratio.

### 2.6 Neural and learned compression

Covered analytically in A12. Data points: NNCP v3.3 compresses
enwik9 to 106,632,363 bytes (0.853 bpb) in 2.8 days on a Ryzen 9
3900 + RTX 4090 (~4.1 kB/s). DZip's honest MDL design (a bootstrap
model trained on the file, parameters stored in the output) beats
gzip by 29%, 7zip by 12%, bsc by 8%, at kB/s-class speeds. TRACE
reproduces at 1.87 bpb (enwik8) / 1.55 (enwik9), ~17 kB/s encode.
"Language Modeling Is Compression" (ICLR 2024) reports Chinchilla
70B at 0.664 bpb on enwik9 with the model excluded; counted, the
model is ~14,008% of the input. A 2026 preprint (Nacrith) reports
0.9389 bpb on enwik8 with ~500 MB of uncounted weights — preprint,
unverified; the pattern (ratio quoted, weights uncounted) is exactly
what the accounting rule exists to reject.

### 2.7 The 2024–2026 frontier, briefly

- Hutter/LTCB: the 2026 jump (§2.1) came from small *counted*
  learned components — the frontier is hybrid, not purely neural.
- zstd 1.5.7 (February 2025): up to +30% small-data speed (Silesia
  4 KB blocks 280→310 MB/s; 32 KB 383→458 MB/s) and the patch-from
  speedup in §2.4.
- XZ Utils 5.8.x (2025) and 7-Zip 24.09/25.00 (2024/2025): steady
  speed and threading improvements, no format upheaval (decoder
  speedup percentages available only from secondary summaries).
- Hardware: vendor figures claim very large device rates (NVIDIA
  Blackwell decompression engine, nvCOMP GPU zstd ≈16 GB/s, Intel
  IAA ≈30 GB/s Deflate decompress) — vendor/projected numbers,
  end-to-end system gains far below device rates; not a design input
  for a portable Krell codec beyond noting that FAST-tier algorithms
  (LZ4/Deflate/zstd class) are the ones hardware accelerates.

## Part 3 — Verification of the founding claims

1. **"Best enwik9 ≈ 91.8 MB at ~61 h/GB"** — the numbers are exact
   (91,778,237 bytes; 220,071 ns/B = 61.13 h/GB), with one naming
   correction: that entry (ex-girlfriend 1.0) tops the *LTCB*, whose
   accounting matches KOLLAPSE's rule. It is not the Hutter Prize
   *awarded* record, which is fx2-cmix-T at 100,424,672 bytes
   (2026-07-24) under the prize's stricter time/RAM limits, and it
   is not vanilla cmix (108,244,767 bytes, ≈173 h/GB).
2. **"LZ4 ≈ 2.1× at 675 MB/s compress / 3,850 MB/s decompress"** —
   exact, from the zstd project's lzbench table (Silesia corpus,
   Core i7-9700K, single core). Hardware- and corpus-specific, as
   all such numbers are.

Not independently verified, and flagged wherever used above:
DictForge's gains (preprint), Nacrith (preprint), cmix-views'
90.5 MB (self-report), the HDiffPatch delta-size compilation
(secondary), hardware-accelerator rates (vendor), XZ 5.8 speedup
percentages (secondary), and one 2026 compression-challenge summary
(secondary only). No protobuf-specific dictionary case study was
found; the documented dictionary corpora are JSON, logs, and HTML
headers.
