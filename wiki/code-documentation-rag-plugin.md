# Code Documentation RAG Plugin — HASMO

> Semantic code search for Hermes Agent via local embedding + vector retrieval.
> Profile: **hasmo**

## Goal

Enable Hermes Agent (in the `hasmo` profile) to semantically search a codebase — finding relevant code, documentation, and context using natural language queries — without sending code to external APIs.

## Architecture

```
Codebase (filesystem)
    │
    ├── tree-sitter (syntactic chunking)
    │       ├── Functions, classes, imports split as semantic units
    │       └── NOT fixed token windows — each chunk is structurally coherent
    │
    ├── Ollama + bge-m3 (embedding model)
    │       ├── 567M parameters, FP16
    │       ├── Multi-function: dense + sparse/colbert retrieval
    │       └── Runs on: Radeon 780M iGPU (Vulkan) or CPU fallback
    │
    ├── FAISS or Chroma (vector storage)
    │       ├── Index: chunk embeddings + metadata (file, line, docstring)
    │       ├── Incremental updates via inotify (file watcher)
    │       └── Metric: inner product or cosine
    │
    └── Hermes Agent (query interface)
            ├── Agent queries semantic search
            ├── Top-k chunks retrieved by similarity
            └── Chunks injected as context for LLM
```

## Technology Stack

| Component | Choice | Reason |
|---|---|---|
| Embedding model | **bge-m3** (567M) | Code-trained, multilingual, multi-function retrieval (FlagOpen/BAAI) |
| Inference host | **Ollama** | Local, Vulkan support, easy model management |
| Vector DB | **FAISS** (or Chroma) | Fast, local, no server needed |
| Chunking | **tree-sitter** | Syntactic — functions, classes, imports stay coherent |
| File watcher | **inotify** | Incremental index updates on file change |
| Profile | **hasmo** | Isolated Hermes profile for HASMO experiments |

## Why bge-m3?

- Explicitly trained on code + natural language pairs (BAAI's MIRACL training)
- Multi-function: supports dense retrieval + sparse (ColBERT-style) in one model
- SOTA on multilingual retrieval (MIRACL, MKQA) — handles docs in any language
- Ollama has it built-in: `ollama pull bge-m3`
- 567M params — runs on 780M iGPU (Vulkan) with ~4-6GB VRAM

**Alternatives considered:**
- `mxbai-embed-large` — stronger general MTEB, but not code-specific
- `nomic-embed-text` — fast, tiny, but mediocre on code
- `embeddinggemma` — Google, code-trained, but requires Ollama v0.11.10+
- API options (voyage-code-2, text-embedding-3-small) — higher quality, but goes external

## Implementation Phases

### Phase 1: Core Index (this session)
- [ ] `ollama pull bge-m3`
- [ ] Chunking pipeline with tree-sitter (Python)
- [ ] Embedding generation via Ollama API
- [ ] FAISS index construction
- [ ] Basic query CLI (pass query → get top-k chunks)

### Phase 2: Incremental Updates
- [ ] inotify file watcher for watched directories
- [ ] Diff-based re-indexing (Merkle tree approach from Cursor)
- [ ] Embedding cache (by chunk content hash)
- [ ] Background indexing with progressive availability

### Phase 3: Hermes Integration
- [ ] Expose as Hermes skill/tool
- [ ] Inject top-k chunks into LLM context
- [ ] Profile-specific config (hasmo profile)
- [ ] Self-querying: agent decides when to search, what to search

### Phase 4: Optimizations
- [ ] Sparse retrieval complement (bge-m3's ColBERT mode)
- [ ] Reranker: BAAI/bge-reranker-v2-gemma (or lightweight variant)
- [ ] Dimension truncation (MRL: 1024 → 512 dims, faster search)
- [ ] Query complexity routing (simple queries → FAISS, complex → LLM re-rank)

## Key Design Decisions

### Syntactic Chunking over Token-Window
Fixed token windows break semantic coherence — a function's docstring, signature, and body should stay together. tree-sitter parses the AST and splits at structural boundaries (function defs, class defs, import blocks, etc.).

### Merkle Tree for Incremental Updates
Inspired by Cursor's secure codebase indexing:
- File change → recompute only that file's chunks → update affected index entries
- Unchanged chunks reuse cached embeddings
- Background indexing so agent doesn't wait for full rebuild

### Local-Only, Zero API Cost
No embeddings going to OpenAI, Cohere, or any external service. Code stays local. This matters for:
- Privacy (code never leaves machine)
- Cost (free at scale)
- Latency (no network round-trip for embedding)

### Ollama + Vulkan on 780M
The 780M iGPU handles bge-m3 comfortably. This leaves the RX 7900 XTX free for the main LLM (Gemma 4 31B via llama.cpp) and eGPU dock bandwidth for inference, not embedding.

## File Structure (planned)

```
~/projects/hasmo/
├── wiki/
│   └── code-documentation-rag-plugin.md   # This document
├── src/
│   └── rag/                               # RAG plugin source
│       ├── __init__.py
│       ├── chunker.py                    # tree-sitter syntactic chunking
│       ├── embedder.py                   # Ollama + bge-m3 interface
│       ├── indexer.py                    # FAISS index build + update
│       ├── watcher.py                    # inotify file watcher
│       ├── query.py                      # Retrieval API
│       └── merkle.py                     # Content-hash tree for diffing
├── scripts/
│   ├── index-codebase.py                 # Full index build
│   └── query-rag.py                      # CLI query tool
└── tests/
    └── test_chunker.py                   # Chunking correctness
```

## References

- [BGE-M3 (FlagOpen/FlagEmbedding)](https://github.com/FlagOpen/FlagEmbedding)
- [BGE-M3 Ollama](https://ollama.com/library/bge-m3)
- [Cursor Secure Codebase Indexing](https://www.cursor.com/blog/secure-codebase-indexing)
- [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard)
- [FAISS](https://github.com/facebookresearch/faiss)
- [tree-sitter](https://tree-sitter.github.io/tree-sitter/)
