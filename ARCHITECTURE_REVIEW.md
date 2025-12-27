# Architecture Review: The Cortex

## Summary

Reviewed: ARCHITECTURE.md for RAG-based LLM engine for AI-Dungeon style DnD5e roguelike.
Target: 16GB VRAM, local LLM inference.

**Verdict**: Feasible. Core spreading activation RAG concept is solid. Model selection requires careful consideration for context budget.

---

## 1. Hardware Feasibility & Model Selection

### Context Budget Problem

With a 4k context window, the budget splits approximately:
- ~2500 tokens for context (facts, world state, characters)
- ~1500 tokens for response

**This is insufficient for complex fantasy scenarios.** A typical scenario requires:

| Context Component | Token Estimate |
|-------------------|----------------|
| System prompt + response schema | ~300-500 |
| Current event description | ~100-200 |
| World state (time, location, weather) | ~50-100 |
| Active characters (2-4 NPCs) | ~200-400 |
| Relevant facts from knowledge graph | ~500-1500 |
| Dialogue history (last 2-3 turns) | ~300-600 |
| **Total context needed** | **~1500-3300** |
| Reserved for response | ~1000-2000 |
| **Minimum practical context** | **~3000-5000** |

For complex scenarios (combat with multiple participants, revelation moments, branching dialogue):
- Need ~4000-6000 tokens context
- Need ~1500-2500 tokens response
- **Minimum practical window: 6k-8k tokens**

### Model Comparison (16GB VRAM)

| Model | Params | Q4_K_M Size | Max Practical Context | Quality |
|-------|--------|-------------|----------------------|---------|
| Qwen2.5-14B | 14B | ~8.5GB | 4-6k (tight) | High |
| Qwen2.5-7B | 7B | ~4.5GB | 8-16k (comfortable) | Medium-High |
| Ministral-8B | 8B | ~5GB | 8-32k (comfortable) | Medium-High |
| Ministral-3B | 3B | ~2GB | 16-64k (very comfortable) | Medium |

### Recommendation: Ministral-8B or Qwen2.5-7B

**Why not Qwen2.5-14B?**
- At Q4_K_M (~8.5GB), leaves only ~7.5GB for KV cache
- 8k context KV cache needs ~3GB, leaving ~4.5GB headroom
- Complex scenarios will hit VRAM ceiling, causing swap to RAM (5-20x slowdown)

**Why Ministral-8B?**
- Native 256k context window (can use 16-32k comfortably)
- Q4_K_M fits in ~5GB, leaves ~11GB for KV cache
- 16k context with 8B model: ~2.5GB KV cache
- **Total: ~7.5GB with 16k context - plenty of headroom**
- Apache 2.0 license, optimized for edge deployment

**Why Qwen2.5-7B as alternative?**
- Strong multilingual support (29+ languages)
- Excellent at structured JSON output
- Good balance of quality vs context capacity

### VRAM Budget (Recommended Setup)

| Component | VRAM (Ministral-8B Q4_K_M) |
|-----------|---------------------------|
| Model weights | ~5GB |
| KV cache (16k context) | ~2.5GB |
| KV cache (32k context) | ~5GB |
| Overhead/buffers | ~1GB |
| **Total (16k ctx)** | **~8.5GB** |
| **Total (32k ctx)** | **~11GB** |

**Assessment**: 16k context is comfortable. 32k possible. This solves the context budget problem.

---

## 2. Core Architectural Concepts

### 2.1 Spreading Activation RAG

The knowledge graph uses spreading activation to find relevant facts. This is the right approach for:
- Limited context windows (retrieves only what's needed)
- Associative memory (finds related facts through connections)
- Emergent narrative (unexpected connections surface naturally)

**Key insight**: This is not a traditional vector-based RAG. Tag associations form a semantic graph where energy flows from trigger tags to related concepts. The architecture correctly prioritizes graph traversal over embedding similarity.

### 2.2 Event-Driven Pipeline

```
GameEvent → ContextAssembler → LLM → ToolExecutor → Response
                ↓                        ↓
           KnowledgeGraph ←──────────────┘
```

**Strength**: Clean separation. The core processes events without controlling game flow.

**Consideration**: Async boundary with Bevy needs careful handling. Use `bevy_tokio_tasks` or channels rather than blocking runtime.

### 2.3 Typed Response System

The architecture defines typed responses (`DialogueResponse`, `CombatResponse`, `DescriptionResponse`) rather than free-form text.

**Benefits**:
- Structured output for game integration
- JSON schema guides LLM output
- Validation catches malformed responses

**Consideration**: Add fallback to graceful degradation. When LLM returns invalid JSON, fall back to templated responses rather than crashing.

---

## 3. Architectural Gaps

### 3.1 Token Budget Management

**Missing**: No mechanism to ensure assembled context fits in context window.

**Solution concept**:
1. Track approximate token count during context assembly
2. Prioritize facts by activation energy (higher energy = more relevant)
3. Truncate lowest-priority facts when approaching budget
4. Reserve space for response generation

### 3.2 LLM Output Validation

**Missing**: Raw JSON parsing will fail on malformed LLM output.

**Solution concept**:
1. Validate against JSON schema before parsing
2. Retry with simplified prompt on failure
3. Fall back to templated response after N retries

### 3.3 Deterministic Behavior

**Issue**: HashMap iteration order varies between runs.

**Impact**: Same input can produce different spreading activation results.

**Solution**: Use IndexMap for graph structures to ensure reproducible behavior.

---

## 4. Simplification Opportunities

### 4.1 Start Without Localization

The `rust-i18n` integration adds complexity. For MVP:
- English only
- Static strings
- Add i18n via feature flag later

### 4.2 JSON for Storage

Bincode is faster but:
- Not human-readable (hard to debug)
- Version migration is painful
- Saves are read once per session

JSON overhead is negligible for this use case.

### 4.3 Minimal Fact Types

The architecture defines many `FactType` variants. For MVP:
- Start with 2-3 types (Relationship, Event, Trait)
- Add types as needed
- Avoid Generic catch-all (defeats type safety)

---

## 5. Implementation Roadmap

### MVP Path (Recommended)

**Phase 1: Proof of Concept**
- Hardcoded world state (no persistence)
- Simple prompt template
- Ollama chat without tools
- Text output only

**Phase 2: Memory System**
- Knowledge graph with spreading activation
- Single tool: `add_fact`
- Token budget limiting

**Phase 3: Structured Output**
- Typed JSON responses
- Full tool system
- Bestiary loading from TOML/JSON

**Phase 4: Game Integration**
- Bevy ECS integration
- Combat system
- Save/load persistence

---

## 6. Dialogue Branching Design

### Current Architecture (from ARCHITECTURE.md)

The system uses `DialogueResponse` with `player_variants: Vec<DialogVariant>`. AI generates options, player selects one.

```rust
pub struct DialogVariant {
    pub text: String,
    pub tone: EmotionalTone,
    pub consequence_hint: Option<String>,
}
```

### Branching Strategy: AI-Controlled Termination

**Approach**: Let AI decide when dialogue should end, with player override.

1. **Natural endings**: AI returns empty `player_variants` when conversation reaches natural conclusion
2. **Player override**: Always include a "leave/end conversation" option as final variant
3. **Depth limiting**: Track dialogue turn count, bias AI toward conclusion after N turns

```rust
pub struct DialogueState {
    pub turn_count: u32,
    pub max_turns_hint: u32,  // Soft limit, AI can exceed
    pub force_conclusion: bool,  // Hard limit reached
}

pub struct DialogVariant {
    pub text: String,
    pub tone: EmotionalTone,
    pub consequence_hint: Option<String>,
    pub ends_dialogue: bool,  // New: marks this as exit option
}
```

**System prompt injection**:
- At turn N-2: "This conversation is reaching a natural point to conclude. Consider wrapping up."
- At turn N: "Generate final response. Player variants should include conclusion options."

### Branching Complexity

Each dialogue turn generates 2-4 variants. Tree explosion is mitigated by:

1. **No branching memory**: Each response is independent; AI doesn't track "paths"
2. **State in knowledge graph**: Consequences are facts, not branches
3. **Player choice → event → new context**: Linear, not tree

**Example flow**:
```
Turn 1: AI generates 3 options
Player picks option 2
→ GameEvent::DialogueChoice emitted
→ Knowledge graph updated with choice consequence
→ Turn 2: AI sees updated facts, generates new options
```

This is a **state machine**, not a **dialogue tree**. Complexity is O(n) per turn, not O(branches^n).

---

## 7. Content Pipeline

### Strategy: D&D SRD Foundation + AI Enhancement

**Phase 1: Hand-written foundation**
- Core bestiary from D&D 5e SRD (goblins, orcs, dragons, etc.)
- Basic item definitions from SRD
- Initial world facts (locations, major NPCs)

**Phase 2: AI-assisted content**
- Generate unique names for generic entities ("Goblin" → "Skrix the Toothless")
- Generate flavor descriptions from stat blocks
- Create personality traits from creature type
- Generate lore connections from bestiary tags

**Data format**:
```toml
# bestiary/goblin_warrior.toml
[base]
template = "srd:goblin"  # Reference SRD stats

[generation]
name_style = "goblin"
generate_description = true
personality_count = 2

[overrides]
# Only specify deviations from SRD
hp_modifier = 5
```

**AI generation is lazy**: Descriptions generated on first encounter, cached in knowledge graph. Reduces startup time, spreads compute cost.

---

## 8. Summary

### Core Architecture: Sound

The spreading activation RAG approach is well-designed:
- Tags extracted from events → associations traversed → relevant facts collected
- Energy decay ensures focus on directly related content
- Configurable thresholds allow tuning context relevance

### Key Architectural Decisions

| Decision | Recommendation |
|----------|---------------|
| Model | Ministral-8B or Qwen2.5-7B (not 14B) |
| Context window | 16k tokens minimum |
| Dialogue branching | State machine, not tree |
| Content pipeline | SRD foundation + lazy AI generation |
| Turn-based combat | Acceptable (2-5s latency) |

### Implementation Priorities

**Phase 1: Core loop (MVP)**
- Hardcoded world state
- Simple prompt template
- Ollama chat (no tools)
- Text output only

**Phase 2: Memory system**
- Knowledge graph with spreading activation
- One tool (`add_fact`)
- Token budget limiting

**Phase 3: Structure**
- Typed JSON responses
- Full tool system
- Bestiary loading from TOML

**Phase 4: Game integration**
- Bevy ECS integration
- Combat system
- Save/load

### Code Quality Notes

Minor Rust patterns to address during implementation:
- Use `IndexMap` for deterministic iteration in graph traversal
- Use `total_cmp` for f32 sorting (avoid NaN panics)
- Define proper error types (`StorageError`, `ProcessingError`)
- Resolve `FactId` type inconsistency (u32 vs Uuid in different places)
