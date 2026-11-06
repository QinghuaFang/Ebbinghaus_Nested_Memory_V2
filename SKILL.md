# :brain: Ebbinghaus Nested Compression Memory

> Stop your LLM from burning tokens and forgetting everything.

## :book: What is this?

Have you ever chatted with an AI for 50 turns, only for it to forget the core rule you set at the very beginning? Or wasted half of your context window on useless "Hello" and "Thanks"?
This Skill solves that. It gives your conversational AI a **human-like hippocampus**.

## :sparkles: Core Features

- **Nested Compression**: Old memories are projected into lower dimensions, like zipping files.
- **Ebbinghaus Decay**: Useless chit-chat fades away, while `reserve` tagged core rules stay forever.
- **Token Guard**: Hard limits on how many tokens are unpacked per turn. No more bankruptcy!

## :rocket: How to use?

The `Ebbinghaus_Nested_Memory_V2.md` in this repo is the skill itself (i.e. `SKILL.md`). There are three ways to feed it to your Agent:

**Option 1: Install as a Skill in WorkBuddy (recommended, set once and it lasts)**

1. Create a skill directory (pick one):
   - User-level (applies to all projects): `~/.workbuddy/skills/ebbinghaus-memory/`
   - Project-level (current workspace only): `<your workspace>/.workbuddy/skills/ebbinghaus-memory/`
2. Save this file as-is into that directory with the exact filename `SKILL.md` (uppercase).
3. After refreshing/restarting the conversation, whenever a need involves "long-conversation memory, token overflow, forgotten rules, memory compression", the Agent will auto-match and load this skill. You can also explicitly trigger it by saying "use the Ebbinghaus memory skill".

**Option 2: Paste directly into context (works on any Agent / any platform)**

- Paste the full content of `SKILL.md` into the System Prompt, or attach it as a file at the start of each turn (i.e. paste the whole block below).
- It is pure Markdown instructions with zero code dependencies — any instruction-following LLM can read and execute it directly.

**Option 3: Tune the knobs (key parameters, edit directly in the file)**

- `max_unpack_token`: total token budget unpacked into context per turn; adjustable anytime.
- `frozen_token_quota`: max proportion of `max_unpack_token` for frozen archives; default 20%.
- `max_nesting_depth`: max recursion depth of nested compression (also the hard cap on "forgetting rounds").
- `similarity_cutoff`: semantic similarity threshold that triggers decompression.
- `weight_threshold`: weight above which a conflict triggers human adjudication.

**How to protect core memories?**
Tag any long-term rule / persona / setting you never want to forget with `reserve`. They are sent to the `Frozen residual archive` — never decaying, never auto-pruned, only bounded by the frozen quota.

---

# Skill: Ebbinghaus Nested Compression Memory
> Simulates human-like memory with recursive "archive-within-archive" layering, gradual forgetting, selective recall, configurable conflict resolution, subspace projection metaphor, quota guards, two-stage retrieval, and explicit human adjudication signalling.
> This skill manages long-term conversation memory without naively discarding raw context. All global rules can be overridden for individual memory entries or memory clusters.

## Overview
This memory system organises conversation history into recursively nested memory layers, analogous to nested archives. Each layer corresponds to a projection onto a lower-dimensional semantic subspace.
- Inner layers: High-fidelity, recent, fine-grained memory; low compression cost, easy to retrieve.
- Outer layers: Aggregated, coarsened long-term summaries; higher decompression cost, only unpacked when relevant cues appear.
- When a memory fragment is retrieved, its decay timer resets, mimicking the Ebbinghaus forgetting curve.
- `Frozen residual archive`: An isolated read-only subspace for manually pinned backup memories, exempt from automatic decay, compression, merging and pruning — but subject to token quota limits at load time.

## Core Mechanism
### Memory Layer Rules
1. **Short‑term active layer**
   Stores near-original conversation fragments, timestamps, weight scores, tags, and trace pointers. Minimal summarization. This is the default reading space for every new turn.
2. **Intermediate archive layers**
   Periodically pull low-access fragments from the inner layer, compress them into structured summaries, and push them outward. Each compression step performs semantic projection and drops weak components.
3. **Deep long‑term archive layers**
   Multiple intermediate archives are merged into heavier, highly abstracted outer snapshots. These snapshots are not routinely loaded unless strong semantic matching occurs.
4. **Frozen residual archive**
   Separate static storage for entries tagged `reserve`. These entries never decay or get pruned automatically, but they are bounded by a dedicated frozen token quota to prevent unbounded context bloat.

### Forgetting & Refresh
- Every memory entry carries a decay counter aligned with an Ebbinghaus-style attenuation schedule.
- If an entry or archive branch is retrieved (decompressed), its counter resets and the relevant fragment is promoted back toward the inner active layer.
- If an entry remains untouched across successive compression rounds, it will be pushed outward, coarsened repeatedly, and eventually pruned unless marked as `reserve`.
- `max_nesting_depth` enforces a hard upper bound on recursion depth to prevent unbounded archive explosion; this value acts as the user-defined forgetting-round limit.

### Retrieval & Decompression (Two‑Stage Filter Pipeline)
On receiving new user input:
1. **Coarse filter (lightweight preselection)**
   Run keyword / tag matching or lightweight TF-IDF scoring over the full archive index. Select the top 5 candidate archives with the highest preliminary relevance. If the total archive count ≤ 5, skip coarse filtering.
2. **Fine semantic similarity scoring**
   Compute full semantic similarity only over the shortlist produced by the coarse filter.
3. Traverse candidates starting from inner layers outward. Unpack only branches whose similarity exceeds `similarity_cutoff`.
4. Enforce two hard token budgets:
   - Global `max_unpack_token`: total token budget for all unpacked memories injected into context in a single turn. This parameter is manually adjustable by the user at any time.
   - `frozen_token_quota`: the maximum proportion of `max_unpack_token` permitted for frozen archive entries; default = 20%.
5. If loading all qualified frozen entries would exceed the frozen quota, sort frozen candidates by descending weight and load until the quota is exhausted. Entries skipped due to quota are marked `skipped` in the index. If the user invokes a skipped frozen entry explicitly by keyword, load it regardless of the current frozen quota.
6. If the combined candidate pool exceeds `max_unpack_token`, prioritise higher-weight, higher-similarity fragments.
7. Inject the final selected memory fragments into the model context for this turn.

## Default Policy Preset
> Globally active; overrideable per entry, per cluster, or per tag.

### Conflict Resolution
- **Primary rule**: Recency priority. When newer inner memory contradicts an older outer projection, the newer record is treated as ground truth.
- **High‑weight branch**: If either conflicting entry's weight exceeds `weight_threshold`, automatic resolution halts, the memory pipeline pauses, and the conflict enters formal human adjudication.
- **Mandatory adjudication alert**: When entering human adjudication, you MUST output this fixed alert and pause all memory operations (compression, promotion, propagation) until the user provides a resolution:

  ```
  [MEMORY ADJUDICATION REQUIRED]
  The memory pipeline is now PAUSED.
  A conflict needs your decision before any memory operation can continue.

  Conflict type : <recency | high-weight>
  Newer record  : <id> — <one-line summary of the inner / newer memory>
  Older record  : <id> — <one-line summary of the outer / older projection>
  Why paused    : <short reason automatic resolution was halted>

  Please resolve by replying with one of:
    A) Keep the newer record as ground truth (discard the older).
    B) Keep the older record (override recency).
    C) Merge the two — specify how.
    D) Custom resolution — specify exactly.

  No compression, promotion, or propagation will run until you reply.
  ```

  You MAY also emit the same event as structured JSON for downstream tooling:

  ```json
  {
    "event": "memory_adjudication_required",
    "status": "paused",
    "conflict": {
      "type": "<recency|high-weight>",
      "newer": { "id": "<entry_id>", "summary": "<...>" },
      "older": { "id": "<entry_id>", "summary": "<...>" },
      "reason": "<...>"
    },
    "blocked_actions": ["compression", "promotion", "propagation"],
    "awaiting": "user_resolution"
  }
  ```

  After the user replies, apply their chosen resolution, then resume the pipeline.
