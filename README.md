# ath10k CSI Exploration — an LLM-authored knowledge dump

**This entire repository was written by an LLM (Claude, Anthropic), not a human.**

It exists to preserve the findings from a single, long interactive session spent
investigating whether Linksys EA8300 routers (Qualcomm `ath10k`/`ath10k-ct`
chipset family: QCA9888 + QCA4019) can produce usable WiFi Channel State
Information (CSI) for motion/presence sensing.

The human in this session directed the investigation, made all risk/safety
decisions (what to test, on which hardware, when to stop), and gave explicit
permission to publish this writeup. The LLM did the research, wrote the code,
ran the commands, and is the sole author of the text in this repository.
Nothing here should be read as independent human research — treat it as you
would any AI-generated technical document: useful as a starting point and a
record of what was tried, but verify anything load-bearing yourself.

## Why this exists

Public information about getting real CSI data out of `ath10k`-family chips
is scattered across a 2016 mailing list thread, a 2020 GitHub issue, an
unanswered forum question, and one abandoned pull request — and every one of
those threads ends in "nobody knows how" or silence. This repo consolidates
everything one investigation session found, including the dead ends, so the
next person (human or LLM) doesn't have to re-derive it from scratch.

## Start here

- **[RESEARCH.md](RESEARCH.md)** — the full writeup: goal, mechanism, live
  hardware testing, prior art, firmware teardown, architecture identification,
  Ghidra analysis, and where the investigation currently stands.

## What this is *not*

- Not a working CSI-based motion/occupancy sensor. That goal was **not**
  achieved. The root question — does this firmware ever expose real
  per-subcarrier channel data to the host — remains open.
- Not a redistribution of Qualcomm/Candela's proprietary firmware binary.
  This repo describes how to extract and analyze that firmware from hardware
  you own; it does not include the binary itself.
- Not independently verified by a human expert in Xtensa or ath10k firmware
  internals. The architecture identification and function analysis in
  RESEARCH.md are supported by the evidence described there (branch-target
  self-consistency statistics, cross-referenced call targets, Ghidra's
  decompiler output) but have not been reviewed by anyone with specialized
  domain expertise in this exact firmware.

## License / reuse

Text and code snippets in this repository may be reused freely. No warranty
of accuracy — this is a record of one investigation, not a reference manual.
