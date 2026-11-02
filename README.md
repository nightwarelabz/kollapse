# KOLLAPSE

A lossless file encoder written in pure Krell, built on Regenerative
Encoding: wherever possible, do not store the file — store the
shortest description from which the decoder regenerates it.

Three encoder tiers — FAST (LZ77 + FSE), BALANCED (dictionary +
adaptive range coding), and MAX (context mixing) — plus SEED mode
(a registered generator + seed), LIBRARY mode (an index into shared
state, with an optional delta), and DELTA mode against a base file.

Every compression result is reported under one accounting rule:
payload size, header size, and the size of any required shared state
are all stated. A ratio that does not count required shared state is
not a KOLLAPSE ratio.

- Format: `.klp`, specified in [SPEC.md](SPEC.md) (draft v0.1)
- Implementation: pure Krell, built with the Krell v1.1 toolchain;
  byte-identical output on x86_64, AArch64, and RISC-V 64
- Status: specification phase — no codec code yet

## Layout

- `SPEC.md` — the format and codec specification

## License

Apache-2.0 (proposed)
