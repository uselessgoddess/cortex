# The Cortex: Implementation Roadmap

## Executive Summary

This roadmap provides a step-by-step journey to build **The Cortex** from scratch: an AI-driven narrative engine for a D&D 5e roguelike game. Each phase validates core architectural concepts before adding complexity, ensuring the vision described in [ARCHITECTURE.md](./ARCHITECTURE.md) and [ARCHITECTURE_REVIEW.md](./ARCHITECTURE_REVIEW.md) is achievable.

**Core Vision:** State-driven AI narrative generation with spreading activation RAG for context-aware storytelling.

---

## From Zero to Hero: Proof of Concept Journey

This roadmap follows a validation-first approach. Each step proves a critical assumption before building upon it.

### Why This Order?

1. **Phase 1** proves the LLM can generate quality prose locally
2. **Phase 2** proves structured data can be reliably extracted
3. **Phase 3** proves spreading activation RAG improves context relevance
4. **Phase 4** proves D&D mechanics integrate cleanly with AI decisions
5. **Phase 5** proves the architecture integrates into a real game engine

Each phase has a clear "proof point" — a demonstration that validates the concept works before investing in the next layer.

---

## Phase 1: Core Narrative Loop

**Goal:** Player action → AI narrative response

**Validates:** [ARCHITECTURE_REVIEW.md §1 Model Selection](./ARCHITECTURE_REVIEW.md#1-model-selection-qwen3-14b-vs-ministral-3-14b)

### What This Phase Proves

- Local LLM inference meets latency requirements (<3s)
- VRAM budget is feasible on target hardware
- Prose quality is suitable for D&D storytelling
- Response format is usable for game integration

### Key Concepts

| Concept | Architectural Reference | Proof Point |
|---------|------------------------|-------------|
| Model viability | ARCHITECTURE_REVIEW.md §1 | Response latency <3s, VRAM <15GB |
| Narrative generation | ARCHITECTURE.md §1.2.1 | AI produces coherent, contextual prose |
| Event-driven design | ARCHITECTURE.md §1.2.2 | System responds to game events |

### Acceptance Criteria

- [ ] Model loads successfully with target quantization
- [ ] Single-turn narrative generation completes in <3s
- [ ] VRAM usage remains stable during extended sessions
- [ ] Prose quality is subjectively acceptable for D&D narration

### Architectural Foundation

This phase implements the core principle from ARCHITECTURE.md §1.2.1:

> "All AI actions must result from analysis of current world state and accumulated memory."

At this stage, "world state" is minimal (player action + scene description), but the pattern is established.

---

## Phase 2: Structured Output & Fact Extraction

**Goal:** AI extracts and returns structured data alongside narrative

**Validates:** [ARCHITECTURE_REVIEW.md §2-3 Single vs Dual Model, Structured Outputs](./ARCHITECTURE_REVIEW.md#2-architecture-decision-single-vs-dual-model)

### What This Phase Proves

- Two-pass generation (narrative → extraction) is viable
- GBNF grammar enforcement produces valid JSON reliably
- Model intelligence affects extraction quality (not just syntax)
- Combined latency remains acceptable (<6s total)

### Key Concepts

| Concept | Architectural Reference | Proof Point |
|---------|------------------------|-------------|
| Two-pass generation | ARCHITECTURE_REVIEW.md §2 | Narrative + extraction in sequence |
| GBNF constraints | ARCHITECTURE_REVIEW.md §3 | 100% valid JSON output |
| Semantic quality | ARCHITECTURE_REVIEW.md §3 | Extracted facts are narratively relevant |

### Acceptance Criteria

- [ ] GBNF grammar produces syntactically valid JSON 100% of time
- [ ] Extracted facts are narratively significant (>80% useful)
- [ ] Combined latency (narrative + extraction) <6s
- [ ] Schema evolution doesn't break existing workflows

### Architectural Foundation

This phase validates ARCHITECTURE_REVIEW.md §3's core insight:

> "GBNF guarantees syntactic validity, not semantic correctness... Intelligence still matters."

The proof point is demonstrating that fact quality depends on model reasoning, not just grammar enforcement.

---

## Phase 3: Knowledge Graph & Spreading Activation RAG

**Goal:** Context-aware generation using accumulated world knowledge

**Validates:** [ARCHITECTURE.md §2.2.2 Context Assembler](./ARCHITECTURE.md#222-module-context_assembler)

### What This Phase Proves

- Spreading activation retrieves relevant facts based on context
- Token budget management prevents context overflow
- Deterministic traversal produces reproducible results
- Retrieved context improves narrative coherence

### Key Concepts

| Concept | Architectural Reference | Proof Point |
|---------|------------------------|-------------|
| Knowledge graph structure | ARCHITECTURE.md §2.2.1 | Tags, facts, associations work as designed |
| Spreading activation | ARCHITECTURE.md §2.2.2 | Energy propagation surfaces relevant facts |
| Token budgeting | ARCHITECTURE_REVIEW.md §4 | Context stays within model limits |
| Determinism | ARCHITECTURE_REVIEW.md §8 | Same inputs → same outputs |

### Acceptance Criteria

- [ ] Spreading activation retrieves contextually relevant facts
- [ ] AI responses reference past events appropriately
- [ ] Token budget prevents context overflow
- [ ] Same seed + input produces identical results

### Architectural Foundation

This phase implements the core RAG mechanism from ARCHITECTURE.md §2.2.1:

> "The knowledge graph enables situation-dependent context retrieval."

The proof point is demonstrating that encounters with tagged entities (e.g., "knight", "crown") surface relevant historical facts (e.g., "Party is hiding from the Crown").

### Research Validation

The spreading activation approach is grounded in:
- [Spreading Activation for RAG](https://arxiv.org/abs/2512.15922) (December 2025)
- Custom implementation preferred over external databases for determinism and offline operation

See [Appendix C](#appendix-c-decision-research) for detailed analysis.

---

## Phase 4: Game Mechanics & Combat

**Goal:** Integrate D&D 5e SRD combat with AI-driven tactical decisions

**Validates:** [ARCHITECTURE.md §2.1 dnd_rules crate](./ARCHITECTURE.md#21-crate-1-dnd_rules-dnd-5e-srd-mechanics)

### What This Phase Proves

- D&D 5e mechanics can be cleanly separated from narrative logic
- AI can make reasonable tactical decisions given combat state
- Bestiary data format supports encounter generation
- Combat latency remains acceptable (~3-5s per turn)

### Key Concepts

| Concept | Architectural Reference | Proof Point |
|---------|------------------------|-------------|
| Mechanics separation | ARCHITECTURE.md §2.1 | dnd_rules crate is engine-agnostic |
| Creature templates | ARCHITECTURE.md §2.1.2 | Bestiary loads and spawns correctly |
| AI combat decisions | ARCHITECTURE.md §1.2.1 | AI selects reasonable actions |
| Localization | ARCHITECTURE.md §2.1.1 | Multi-language creature names work |

### Acceptance Criteria

- [ ] Core D&D 5e SRD rules implemented correctly
- [ ] AI tactical decisions are reasonable (not random/suicidal)
- [ ] Combat turn latency <5s
- [ ] At least 20 SRD creatures functional in bestiary

### Architectural Foundation

This phase implements the separation described in ARCHITECTURE.md §2.1:

> "This crate contains raw DnD 5e mechanics... It is separate from the game engine (Bevy ECS) and focuses only on rule definitions."

The proof point is running combat encounters where rules are validated by dnd_rules while AI narrates outcomes.

### Data Source Decision

See [Appendix C.1](#c1-bestiary-data-source-open5e-api-vs-handcrafted) for analysis of Open5e API vs handcrafted bestiary data. **Recommendation: Hybrid approach** — handcrafted data with optional Open5e import.

---

## Phase 5: Bevy Integration & Full Game

**Goal:** Complete game engine integration with persistence

**Validates:** [ARCHITECTURE.md §2.1.4 Bevy Integration](./ARCHITECTURE.md#214-integration-with-game-engine-bevy)

### What This Phase Proves

- WorldContext snapshot pattern works for ECS → Cortex communication
- Async LLM calls don't block the game loop
- Save/load preserves complete game state
- Full gameplay loop is cohesive and playable

### Key Concepts

| Concept | Architectural Reference | Proof Point |
|---------|------------------------|-------------|
| ECS integration | ARCHITECTURE.md §2.1.4 | Bevy ↔ Cortex sync works correctly |
| Async patterns | ARCHITECTURE_REVIEW.md §7 | LLM calls don't block rendering |
| Persistence | ARCHITECTURE.md §2.2.1 | Full state saves and loads correctly |
| UI integration | ARCHITECTURE.md §1.2.4 | Dialogue variants display properly |

### Acceptance Criteria

- [ ] Game maintains 60 FPS during LLM calls
- [ ] Save/load produces identical game state
- [ ] Complete gameplay loop (explore → encounter → combat → narrative) functional
- [ ] No memory leaks during extended play sessions

### Architectural Foundation

This phase validates the decoupling principle from ARCHITECTURE.md §1.2.4:

> "The core operates on pure data - it has no knowledge of graphics rendering, input handling, audio systems, network protocols."

The proof point is demonstrating that narrative_core remains completely independent of Bevy internals.

---

## Validation Strategy

### Testing Approach

Each phase includes validation tests that prove the architectural concepts:

| Test Type | Purpose | When to Use |
|-----------|---------|-------------|
| Unit tests | Verify core logic | Every phase |
| Integration tests | Verify component interaction | Phases 3-5 |
| Snapshot tests | Catch unintended changes | LLM prompt templates |
| Property-based tests | Find edge cases | Dice rolling, graph traversal |

### Quality Gates

Before advancing to the next phase:

1. All acceptance criteria must be met
2. Performance targets must be validated
3. Architectural principles must be preserved
4. No regressions in previous phase functionality

---

## Risk Mitigation

### Technical Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| LLM latency exceeds budget | High | Async processing, preloading, fallback quantization |
| VRAM overflow | High | Monitor usage, automatic fallback to Q4_K_M |
| Knowledge graph explosion | Medium | Fact expiration, importance-based pruning |
| GBNF parse failures | Medium | Retry logic, validation layer |

### Dependency Risks

| Dependency | Risk | Mitigation |
|------------|------|------------|
| llama.cpp | Breaking API changes | Pin versions, integration tests |
| Model availability | Model removed from sources | Local storage, alternative models |
| Bevy updates | ECS API changes | Abstraction layer, follow upgrade guides |

---

## Success Metrics

### Phase Completion Criteria

| Phase | Core Proof Point | Must Demonstrate |
|-------|-----------------|------------------|
| 1 | LLM works locally | <3s latency, stable VRAM |
| 2 | Extraction works | 100% valid JSON, relevant facts |
| 3 | RAG works | Context improves responses |
| 4 | Combat works | AI makes sensible decisions |
| 5 | Game works | Full loop is playable |

### Overall Project Success

- **Technical:** All phases complete with validation criteria met
- **Architectural:** Core principles from ARCHITECTURE.md preserved
- **Performance:** Smooth local execution on target hardware
- **Experience:** Engaging AI-driven narrative gameplay

---

## Appendix A: Architecture Reference Map

This section maps roadmap phases to specific architectural documentation.

### ARCHITECTURE.md References

| Section | Relevant Phases | Key Concepts |
|---------|-----------------|--------------|
| §1.2.1 State-Driven Architecture | 1, 3, 4 | AI decisions as pure functions of state |
| §1.2.2 Event-Driven Communication | 1, 5 | Loose coupling, game engine controls flow |
| §2.1 dnd_rules crate | 4 | D&D mechanics separation |
| §2.2.1 knowledge_base | 3 | Spreading activation, tag associations |
| §2.2.2 context_assembler | 3 | Token budgeting, fact prioritization |

### ARCHITECTURE_REVIEW.md References

| Section | Relevant Phases | Key Decisions |
|---------|-----------------|---------------|
| §1 Model Selection | 1 | Ministral-3-14B recommendation |
| §2 Single vs Dual Model | 2 | Single model, two-pass approach |
| §3 Structured Outputs | 2 | GBNF + model intelligence |
| §4 Token Budget | 3 | Context window management |
| §7 Implementation Roadmap | All | Phase definitions and order |

---

## Appendix B: Research Sources

### LLM & Inference

- [llama.cpp](https://github.com/ggml-org/llama.cpp) — Core inference engine
- [Rust bindings for llama.cpp](https://github.com/edgenai/llama_cpp-rs) — High-level Rust API
- [GBNF Grammar Guide](https://github.com/ggml-org/llama.cpp/blob/master/grammars/README.md) — Structured output specification

### Knowledge Graphs & RAG

- [Spreading Activation for RAG](https://arxiv.org/abs/2512.15922) — Academic foundation (December 2025)
- [GraphRAG-RS](https://github.com/automataIA/graphrag-rs) — Rust implementation reference

### Game Development

- [Bevy Engine](https://bevyengine.org/) — ECS game engine
- [Open5e API](https://open5e.com/) — D&D 5e SRD reference
- [The Red Prison](https://store.steampowered.com/app/1074040/The_Red_Prison/) — D&D 5e roguelike reference

### Model Selection

- [Ministral-3-14B](https://huggingface.co/mistralai/Ministral-3-14B-Reasoning-2512) — Recommended model
- [Mistral 3 Announcement](https://mistral.ai/news/mistral-3) — Benchmark data

---

## Appendix C: Decision Research

This section provides in-depth analysis of key architectural decisions.

### C.1 Bestiary Data Source: Open5e API vs Handcrafted

**Question:** Should we use Open5e API or create bestiary by hand for more control?

#### Analysis

| Approach | Pros | Cons |
|----------|------|------|
| **Open5e API** | 3,539 creatures, automatic updates | External dependency, network required, no custom fields |
| **Handcrafted** | Offline-first, full control, AI generation hints | Initial data entry, maintenance burden |
| **Hybrid** | Best of both worlds | Slightly more complexity |

#### Recommendation: Hybrid Approach

Use handcrafted data as the primary source with Open5e as a reference and import tool.

**Rationale:**
- **Offline-first is essential** — Game shouldn't break if external APIs are down
- **Custom fields needed** — AI generation hints (tactical preferences, naming styles) aren't in Open5e
- **SRD is sufficient** — ~300 SRD creatures provides plenty of variety
- **Open5e valuable for reference** — Use as development tool, not runtime dependency

#### Source References
- [Open5e API v2](https://api.open5e.com/v2/) — 3,539 creatures across 71 pages
- [SRD Monster JSON (tkfu)](https://gist.github.com/tkfu/9819e4ac6d529e225e9fc58b358c3479) — Bootstrap data
- [BTMorton dnd-5e-srd](https://github.com/BTMorton/dnd-5e-srd) — SRD in multiple formats

---

### C.2 RAG Database: Chroma vs GraphRAG-RS vs Custom

**Question:** Should we use production-ready RAG database like Chroma, specialized GraphRAG-RS, or custom implementation?

#### Analysis

| Solution | Spreading Activation | Deterministic | Offline | Complexity | Verdict |
|----------|---------------------|---------------|---------|------------|---------|
| **Chroma** | Vector similarity only | No | Requires server | High | Not recommended |
| **GraphRAG-RS** | PageRank-based | No | Heavy dependencies | High | Not recommended |
| **Custom** | Native implementation | Yes (IndexMap) | Yes | Low | **Recommended** |

#### Recommendation: Custom Implementation

Build a purpose-built spreading activation knowledge graph.

**Rationale:**
- **Algorithm match** — The spreading activation paper describes a specific algorithm that's simple to implement
- **Determinism required** — ARCHITECTURE_REVIEW.md specifies IndexMap for reproducible traversal; external solutions don't guarantee this
- **No server dependency** — Games shouldn't require database servers
- **Performance** — In-memory graph with ~1000 facts is trivially fast; vector search is overkill
- **Integration** — Custom implementation integrates directly with game state

#### Source References
- [Chroma](https://www.trychroma.com/) — Vector database (evaluated, not recommended)
- [GraphRAG-RS](https://github.com/automataIA/graphrag-rs) — Rust GraphRAG (evaluated, not recommended)
- [Spreading Activation Paper](https://arxiv.org/abs/2512.15922) — Academic foundation
- [IndexMap](https://docs.rs/indexmap/) — Deterministic hash map for Rust

---

*This roadmap is a living document. Update as implementation progresses and new insights emerge.*
