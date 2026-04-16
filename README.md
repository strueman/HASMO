# HASMO

**Hermes Agent Small Model Offloading** — Compute optimization and efficiency research project.

## Overview

HASMO explores which Hermes agent tasks could be handled by a small, fast, cheap model running as a preprocessing or filtering layer — distinct from the main agent model. The goal is to reduce API costs, conserve context window space, and improve latency by handling routine filtering, extraction, and triage tasks at the small model layer.

## Status

**Active research** — architectural analysis complete, implementation not yet started.

## Key Documents

- [Architectural Analysis](./wiki/small-model-offloading-architectural-analysis.md) — Full analysis of 12 candidate tasks across 3 tiers
- [Wiki Index](./wiki/index.md) — Project wiki

## Quick Summary

**Option B (preprocessor) pattern** is preferred over Option A (proxy):

```
Tool Output → Small Model (filter/distill) → Main Model input
```

The small model trims fat from tool outputs before they hit the main model. Main model retains full control.

**Tier 1 candidates** (highest value, lowest risk):
- Web search result filtering — 60-80% token savings
- Web page content extraction — 10-50x data reduction
- Terminal output triage — build logs become structured results
- Session history pruning — selective compression between major steps

See the full architectural analysis for all 12 tasks, implementation phases, and open questions.

## Hardware Context

Running on a Minisforum UM890 Pro (Ryzen 9 8945HS + Radeon 780M iGPU) with AMD Radeon RX 7900 XTX 24GB via OCuLink. The 780M iGPU (~48GB accessible from 64GB shared LPDDR5) is available to run small models locally at low cost.

**Small model candidates:** Gemma 4 E2B Q4 (~1.5GB, ~10-15 tok/s on 780M Vulkan)

## Related

- [Hermes Agent](https://github.com/nousresearch/hermes-agent) — Nous Research's self-improving CLI/messaging AI agent
- [llm-wiki skill](../wiki/queries/llm-wiki-architectural-analysis.md) — Wiki integration for Hermes
