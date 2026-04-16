# Small Model Offloading: Architectural Analysis for Hermes Agent

**Date:** 2026-04-15  
**Author:** Simon + Hermes Agent  
**Status:** Brainstorming / Architectural Planning

---

## Overview

This document analyzes which Hermes agent tasks could be handled by a small, fast, cheap model ("small model") as a preprocessing or filtering layer — distinct from the main agent model. The goal is to reduce API costs, conserve context window space, and improve latency by handling routine filtering, extraction, and triage tasks at the small model layer before data reaches the main agent.

**Context:** Simon runs a Minisforum UM890 Pro (Ryzen 9 8945HS + Radeon 780M iGPU) with an AMD Radeon RX 7900 XTX 24GB via OCuLink. The 780M iGPU (~48GB accessible from 64GB shared LPDDR5) is available to run small models locally at low cost.

---

## The Two Architectural Patterns

Before diving into the task analysis, there are two fundamental patterns:

### Option A: Proxy Mode
All requests flow through the small model first. The small model decides whether to call tools, refine queries, route to subagents, or pass to the main model.

```
User → Small Model → [Tool | Main Model | Skip]
```

**Pros:** Powerful, can reduce unnecessary work entirely.  
**Cons:** Broad scope for failure; adds latency to everything; requires reliable routing logic.

### Option B: Preprocessor Mode *(Preferred)*
The small model operates between tool output and main model input. The main model still controls everything; the small model just trims the fat.

```
Tool Output → Small Model (filter) → Main Model input
```

**Pros:** Simple, additive (can't make things worse, only better), easy to measure.  
**Cons:** Can't reduce upstream waste (e.g., redundant API calls before tools run).

**Recommendation:** Start with Option B for data-heavy tools (search, crawl, terminal). Prove the value. Then consider Option A for task routing if the pattern holds.

---

## Task Tiers

Tasks are classified into three tiers based on implementation complexity and expected value:

| Tier | Characteristics | Examples |
|------|----------------|----------|
| **Tier 1** | High value, clear scope, well-defined filtering task, low risk | Web search filtering, content extraction, terminal triage |
| **Tier 2** | Valuable but requires more care; either riskier or more complex | Task decomposition, skill selection, code diff analysis |
| **Tier 3** | Speculative; uncertain value, needs more research | Search query refinement, multi-model result merging |

---

## Tier 1: High-Value Offloads

### 1. Parallel Web Search Result Filtering

**What it does:**  
Deduplicates across result sets, scores relevance to the query, strips boilerplate, keeps unique information. Returns structured JSON rather than raw search results.

**Example:**
```json
// Input: 8 parallel search queries, 10 results each (80 results, ~4000 tokens)
// Output:
{
  "query": "gemma 4 speculative decoding llama.cpp",
  "results": [
    {
      "url": "https://reddit.com/r/LocalLLaMA/...",
      "relevance_score": 0.95,
      "key_finding": "+29% avg speedup with E2B draft model",
      "unique_info": "Draft-max 8 is the sweet spot; vocab mismatch was the initial blocker"
    },
    ...
  ],
  "deduplicated_count": 80,
  "returned_count": 12
}
```

**Saves:** 60-80% of search result tokens. On an 80-result search, that's significant.

**Implementation:** llama.cpp served on 780M, or cheap API call. E2B Q4 (~10-15 tok/s on 780M) handles text filtering in milliseconds.

**Risks:** Small model might miss nuanced relevance. **Mitigation:** include a flag so the main model can request full results if needed.

---

### 2. Web Page Content Extraction

**What it does:**  
Takes raw crawled page + query context. Extracts only relevant passages, optionally summarizes each section.

**Example:**
```json
// Input: 5000 tokens of raw crawled markdown
// Output:
{
  "relevant_sections": [
    {
      "section": "Speculative Decoding Performance",
      "relevance": "high",
      "content": "Gemma 4 31B with E2B draft achieves +29% average speedup...",
      "original_length": 1200,
      "extracted_length": 180
    }
  ],
  "summary": "The page confirms draft-max 8 as optimal. LocalLLaMA benchmarks show...",
  "full_content_available": true
}
```

**Saves:** Context window space, token cost, and — critically — keeps the main model's context for synthesis rather than absorption. Crawled pages are often 10-50x larger than what matters.

**This is more valuable than search filtering** because the data reduction ratio is higher.

**Risks:** Small model might cut context the main model would have used. **Strong mitigation:** include a `"full_content_available"` flag so the main model can request more if needed.

---

### 3. Session History Pruning (Between Major Steps)

**What it does:**  
After a complex multi-step task, the small model summarizes completed steps, identifies unresolved items, and distills key facts. The distilled summary replaces full history in the main model's context.

**This is different from Hermes's current context compression** because it's selective — it keeps what's relevant to the ongoing task rather than just chunking and compressing uniformly.

**Example:**
```json
{
  "completed_steps": [
    "Investigated speculative decoding options for Gemma 4",
    "Benchmarked E2B draft model on 780M — baseline 5 tok/s",
    "Identified vocab mismatch issue with early GGUF downloads"
  ],
  "key_facts": [
    "E2B is the better draft model than E4B (smaller, faster, sufficient quality)",
    "26B-A4B MoE gets less benefit from spec dec due to expert activation pattern",
    "N-gram spec dec shows 0% improvement on consumer GPUs with diverse workloads"
  ],
  "pending": [
    "Run spec dec benchmark comparison on 7900 XTX",
    "Test E2B draft with Gemma 4 31B Q4 on pooled VRAM"
  ]
}
```

**Risks:** Highest risk of Tier 1 — summarization can lose nuance. **Mitigation:** Use only between major workflow steps (not mid-task), and preserve raw history in a side channel for recovery.

---

### 4. Terminal Output Triage

**What it does:**  
Parses stdout/stderr, classifies outcome (success / failure / warning), extracts the relevant error message if any.

**Example:**
```json
// Input: 500 lines of build output
// Output:
{
  "status": "failure",
  "summary": "TypeError in src/agents/hermes.py line 42",
  "relevant_lines": [
    "ERROR: TypeError: cannot concatenate 'str' and 'NoneType'",
    "File 'src/agents/hermes.py', line 42, in process_message",
    "  result = prefix + message  # line 42"
  ],
  "warning_count": 3,
  "error_count": 1,
  "warnings_summary": "DeprecationWarning for asyncio.Event.set()"
}
```

**Saves:** Build logs are frequently 10-50KB. The main model seeing that in context is pure waste — it just needs to know if it worked and what to fix.

**Risks:** Very low. Terminal output is structured enough that a small model handles it reliably.

---

## Tier 2: Medium-Value Offloads

### 5. Task Decomposition (Pre-Filtering)

**What it does:**  
Breaks a complex user request into subtasks, identifies dependencies, estimates what tools are needed, and routes each to the appropriate skill/tool.

**Latency cost:** Small — runs before the main model would start thinking anyway.

**Example:**
```json
// Input: "I need to set up speculative decoding for Gemma 4, benchmark it against my baseline, and update the wiki with findings"
// Output:
{
  "subtasks": [
    {
      "task": "Configure spec dec with E2B draft on 7900 XTX",
      "tool": "terminal",
      "context_needed": "Model paths, llama-server location",
      "blocking": []
    },
    {
      "task": "Run token throughput benchmark",
      "tool": "terminal",
      "context_needed": "llama-bench command, output parsing",
      "blocking": ["Configure spec dec"]
    },
    {
      "task": "Document findings in wiki",
      "tool": "file/write",
      "context_needed": "Benchmark results, wiki path",
      "blocking": ["Run benchmark"]
    }
  ]
}
```

**Risks:** Small model might decompose incorrectly. **Mitigation:** main model can override/revise the decomposition.

---

### 6. Skill and Context Selection

**What it does:**  
At session start (or when a new task begins), the small model reads the request and decides which skills and wiki pages are relevant — before loading everything.

**Example:**
```json
// Input: "Benchmark Gemma 4 31B spec dec on my setup"
// Output:
{
  "relevant_skills": [
    {"name": "llama-cpp", "relevance": "high", "reason": "llama-server commands, Vulkan flags"},
    {"name": "systematic-debugging", "relevance": "medium", "reason": "benchmarking methodology"}
  ],
  "relevant_wiki_pages": [
    {"path": "queries/amd-7900-xtx-780m-um890-pro-setup", "relevance": "high"},
    {"path": "concepts/speculative-decoding", "relevance": "high"}
  ],
  "skip_skills": ["obsidian", "linear", "xitter"]
}
```

**Saves:** Not tokens in the main model context, but LCM processing time and tool initialization overhead.

**Risks:** Medium — might skip a skill the main model ends up needing, causing back-and-forth.

---

### 7. Code Diff Analysis

**What it does:**  
Summarizes a git diff: what changed, whether it looks reasonable, whether tests pass, what the key risk areas are.

**Example:**
```json
{
  "summary": "Refactors speculative decoding configuration into a separate module",
  "change_type": "refactor",
  "risk_level": "low",
  "areas_of_concern": [
    "Draft model GPU routing changed — verify -ngld flag still works"
  ],
  "test_status": "passing",
  "test_changes": "Added 3 new benchmark tests for spec dec flags"
}
```

**Risks:** Low. Diff analysis is structured and the output is additive — the main model can read more if needed.

---

### 8. Cron Job Output Filtering

**What it does:**  
Reads cron job output, classifies: ignore / flag for review / act now, and drafts a response if acting.

**Example:**
```json
{
  "action_required": "act_now",
  "summary": "Daily benchmark run completed. Gemma 4 31B: 57.2 tok/s (vs baseline 57.1). No significant deviation.",
  "draft_response": "Benchmark run complete. Results within normal range. No action needed."
}
```

**Saves:** Avoids waking the main agent for routine/no-op cron runs.

**Risks:** Low — false negatives are recoverable when the main agent does wake up.

---

## Tier 3: Speculative

### 9. Search Query Refinement (Pre-Search)
Generates 3-5 targeted search queries from a vague user request. Could reduce total searches needed. **Risks:** Might make things worse if the small model misreads intent.

### 10. Commit Message / Changelog Generation
Writes conventional commit messages from diffs. Low risk — Gemma 4 E2B handles this well and mistakes are easily fixed.

### 11. Structured Data Extraction from Files
Extracts relevant rows/fields from JSON/CSV/YAML based on task description. Worth considering for large data files.

### 12. Multi-Model Result Merging
Synthesizes parallel subagent outputs — identifies consensus, flags disagreements, recommends next steps. Interesting for subagent workflows.

---

## What Stays with the Main Model

These should **never** be offloaded to the small model:

- Actual reasoning — anything requiring genuine chain-of-thought
- Code implementation — writing, reviewing, debugging
- Complex planning — multi-step plans with non-obvious dependencies
- User communication — responding to the user
- Skill implementation decisions — choosing how to do something
- Novel problem solving — anything outside established patterns
- Security/policy decisions — approving or blocking actions
- Quality-critical summarization — where nuance directly impacts outcomes

---

## Hardware Considerations for Simon's Setup

The 780M iGPU is the natural home for the small model:

| Model | Size | Speed (780M Vulkan) | Use Case |
|-------|------|---------------------|----------|
| Gemma 4 E2B Q4 | ~1.5GB | ~10-15 tok/s | Fast filtering, pre-filtering |
| Qwen 2.5 3B Q4 | ~2GB | ~8-12 tok/s | Slightly smarter filtering |
| Phi-4-mini Q4 | ~2.5GB | ~8-12 tok/s | Reasoning-adjacent filtering |

For this use case, **E2B is the right pick** — it's fast, small, handles text well, and is already available for speculative decoding use.

**VRAM:** With the 7900 XTX handling the main model (31B Q4 ~18GB), the 780M has ~30GB headroom for the small model. No contention.

**Serving:** The small model runs as a separate llama-server instance on the 780M. The main agent calls it via localhost API — same pattern as using OpenAI-compatible APIs.

---

## Implementation Priorities

### Phase 1: Terminal Output Triage + Search Result Filtering
Easiest to implement, lowest risk, clear value. Start here.

### Phase 2: Web Page Content Extraction
Higher value than search filtering. More complex but still well-scoped.

### Phase 3: Task Decomposition + Skill Selection
More ambitious. Requires the main model to trust the small model's routing.

### Phase 4: Session History Pruning
Highest value but highest risk. Only after Phases 1-3 prove the pattern.

---

## Open Questions

1. **API interface:** Does the small model run as a separate service (llama-server on 780M) or as an embedded subprocess? Separate service is simpler but adds network hop.

2. **Fallback behavior:** When the small model is uncertain, does it pass through unfiltered or abstain and let the main model handle it?

3. **Trust model:** Does the main model see the small model's intermediate outputs (for interpretability) or only the final filtered result?

4. **Cost tracking:** Should small model token usage be tracked separately from main model for cost attribution?

5. **Multi-model routing:** If running multiple subagents in parallel, does each get its own small model instance or a shared one?

---

## Summary

| Tier | Task | Value | Risk | Priority |
|------|------|-------|------|----------|
| 1 | Web Search Result Filtering | High | Low | Phase 1 |
| 1 | Web Page Content Extraction | Very High | Low | Phase 2 |
| 1 | Session History Pruning | High | Medium | Phase 4 |
| 1 | Terminal Output Triage | Medium | Very Low | Phase 1 |
| 2 | Task Decomposition | Medium | Medium | Phase 3 |
| 2 | Skill/Context Selection | Medium | Medium | Phase 3 |
| 2 | Code Diff Analysis | Medium | Low | Phase 3 |
| 2 | Cron Output Filtering | Medium | Low | Phase 2 |
| 3 | Search Query Refinement | Medium | Medium | TBD |
| 3 | Commit Message Generation | Low | Very Low | TBD |
| 3 | Structured Data Extraction | Medium | Low | TBD |
| 3 | Multi-Model Result Merging | Medium | Medium | TBD |
