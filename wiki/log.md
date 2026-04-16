# HASMO Project Log

## [2026-04-16] Code Documentation RAG Plugin Started

- Architectural decision: pure local RAG (zero API cost)
- Embedding model: **bge-m3** via Ollama (567M, code-trained, multilingual, multi-function retrieval)
- Stack: Ollama + bge-m3 + FAISS + tree-sitter + inotify
- Profile: **hasmo** (isolated Hermes profile)
- Plan: Phase 1 (core index) → Phase 2 (incremental updates) → Phase 3 (Hermes integration) → Phase 4 (optimizations)
- Wiki doc created: `code-documentation-rag-plugin.md`

## [2026-04-16] Project Created

- GitHub repo created: https://github.com/strueman/HASMO
- Hermes profile created: `hasmo` at `~/.hermes/profiles/hasmo`
- Wiki path set to: `~/projects/hasmo/wiki`
- Architectural analysis document filed from `~/small-model-offloading.md`
- SSH GitHub auth configured (key already present on GitHub)
