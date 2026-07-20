# GemmaEdgeV2 — Sovereign Local AI Orchestration Engine

Fully offline, multi-model LLM orchestration concept. No cloud APIs, no telemetry, no external inference calls — routing, embedding, auditing, and generation are all designed to run on local hardware.

## Status / What this is

This is a design blueprint, not a code drop. It documents an architecture — routing logic, design decisions, thresholds, and the reasoning behind each one — for a multi-model orchestration system built and tested for personal use. No source is included: no selector implementation, no module prompts, no API or frontend code. What follows describes *how the system is designed to behave*, not a walkthrough of files you can open and run.

If you're trying to build something similar, this describes the routing and VRAM-management approach in enough depth to design your own version. If you're trying to reproduce it wholesale, you'd be starting from this reasoning and writing every line yourself — nothing here is copy-paste-able.

**A note on the missing scripts:** names like `misha_selector.py`, `misha_cold_swap.py`, `misha_fallback_selector.py`, `personality_matrix.json`, `fallback_insta_trigger.json`, and `toxic_reference_examples.json` appear below as *labels for where a given piece of logic conceptually lives*, not as files you'll find in this repo. The same goes for the API and frontend layers described further down. If something here doesn't fully make sense without seeing the actual implementation, that's expected — open an issue if you want to talk through the reasoning further.

Built and tested on: NVIDIA RTX 5060 Ti (8GB VRAM). Cold-swap policy: one active GGUF model in VRAM at a time.

## Target audience

This assumes you already know:
- Python
- LLM inference basics (tokenization, context windows, quantized model formats)
- REST APIs and async request handling
- Vector embeddings and cosine similarity

It is NOT a beginner introduction to LLM orchestration or agent routing.

## Core Idea

A single user query is never sent straight to one fixed model. It passes through a routing layer that decides *which* specialist model should handle it, then loads only that model into VRAM — purging whatever was loaded before.

## Routing Logic

The routing decision is designed to happen in strict order, first match wins:

1. **Instant-trigger phrases** — a dedicated phrase list, kept entirely separate from the semantic routing data. Any match forces an immediate bypass to the MainCore module, skipping all further scoring — no embedding step, no auditor model load, cheapest possible path for known-bad input.
2. **Semantic matrix scan** — the query is embedded (local MiniLM-class sentence encoder) and compared against per-module trigger-phrase embeddings using a hybrid cosine score:

   `hybrid_score = (max_cos * max_weight) + (mean_cos * mean_weight)`

   Weights used in testing: `max_weight: 0.95`, `mean_weight: 0.05` — the single closest match matters far more than the average similarity across all trigger phrases. Highest-scoring module wins the initial candidacy.
3. **Toxicity audit** — designed to run **unconditionally on every input that clears step 1**, regardless of how confident the semantic match was. This was a deliberate correction after testing showed a real gap: when the audit only ran on *low*-confidence matches, a toxic query that happened to score high on a legitimate module's trigger phrases (an insult wrapped in a technical-sounding prefix, for example) skipped the toxicity check entirely. Two independent signals run in parallel:
   - **Model signal**: a multi-label toxicity classifier scoring several categories independently (general toxicity, severe/targeted toxicity, obscenity, threats, insults, identity-based hate), each against its own threshold.
   - **Reference-embedding signal**: cosine similarity between the query and a curated set of example phrases per category — independent of the classifier's own learned weights, catching near-identical phrasing even where the model itself under-scores it.
   
   Either signal crossing its threshold overrides the routing decision: severe categories escalate to a hard security response; general categories escalate to a softer buffered response. If neither signal is available (model failed to load), the design currently defaults to routing everything to MainCore rather than proceeding — see Known Limitations for why that's a real risk, not a safe default.
4. **Hard prefixes** — an explicit prefix (naming a target module directly) forces that target, *if and only if* the audit in step 3 didn't already override it. A clean prefix wins; a toxic one gets quarantined regardless of what prefix it's wearing.
5. **Direct name-call** — casually addressing the system by name routes to the Social module, name stripped from the payload — same audit-gate applies as step 4.

Once a target module is resolved, the swap layer purges the currently loaded model (if different) and loads the new one.

## VRAM Management

The design keeps a single active model tracked at a time: swapping models purges the outgoing one's weights and forces a full memory release before loading the next — never two GGUF models held in VRAM simultaneously. A process-exit hook is designed to guarantee a full purge even on abnormal shutdown.

**Why this needs a direct CUDA-compiled Python binding, not a wrapper like Ollama:** this kind of cold-swap depends on being able to close a specific loaded model instance and force immediate memory release, on your own schedule. That level of control means calling CUDA-compiled bindings directly — you're issuing the load/unload yourself, not asking a separate server process to manage it on its own internal keep-alive/eviction logic. Route the same behavior through a wrapper runtime and you lose that control: you're back to trusting someone else's process-management decisions instead of owning the swap yourself.

## Token Budget

Per-module token limits are designed to live in one config source. Before generation, the full prompt (system + history + new input) is meant to be tokenized against the active model's own tokenizer and checked against that module's limit — not an approximate character count.

## Specialist Modules

Each module is conceived as a self-contained unit: its own model path, its own generation parameters, its own system prompt.

| Module | Model class | Context | GPU layers | Token limit | Role |
|---|---|---|---|---|---|
| **MainCore** | ~2B, quantized | 1024 | 32 | 2048 | Gateway / buffer for unrouted or low-confidence input. Hard security refusal on flagged threats. Points users to specialist prefixes. |
| **Social** | ~2B, quantized | 16384 | 26 | 4096 | Casual interaction, personality-forward, no visible reasoning blocks. |
| **Sovereign** | ~2B, quantized | 16384 | 26 | 4096 | Meta-level system identity — explains the architecture itself, locked to the canonical module names only. |
| **Reasoning** | ~4B, quantized | 16384 | 64 | 16384 | Transparent step-by-step logic chain, fixed output format. |
| **Auditor** | ~4B, quantized | 16384 | 64 | 8192 | Career/CV/contract analysis — reframes raw engineering work into strategic/architectural language. Fixed 4-part output structure. |
| **Oracle** | ~4B, quantized | 16384 | 64 | 8192 | Probability/RNG reasoning (gacha pity, odds, risk framing), fixed structured output. |

Lighter models get a smaller GPU-layer allocation; heavier reasoning-class models get more, reflecting their heavier context/task load.

## Memory Layer

Each specialist is designed to keep an isolated memory index — one module's recall is never visible to another. Embeddings reuse the same local sentence encoder used for routing, live entirely in RAM, and never touch VRAM. Retrieval is scoped to a single module's index at a time; results are meant to be rendered as a labeled block injected into that module's next prompt.

## Config

A single config source is meant to hold:
- Local model snapshot paths — offline mode, CPU-only for the auxiliary models
- Router thresholds, hybrid-score weights, and separate paths for the three independent trigger-phrase/reference data files described above — kept apart deliberately, never merged into one structure
- Per-label toxicity thresholds and severe/general category groupings — tunable without touching code
- Per-module paths and token limits
- Hardware VRAM limit and cold-swap behavior
- API and frontend ports

## API Layer (conceptual)

Designed as a FastAPI app with a single shared selector instance held in application state — the selector, its memory, and its session history persisting for the life of the process.

**Routes, as designed:**

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Basic liveness check. |
| `/status` | GET | Full telemetry snapshot: VRAM/GPU/CPU/RAM usage, active module, token limit, session length, memory count. |
| `/ping` | GET | Telemetry only, no session/module info. |
| `/tokens` | GET | Per-module session length + token limit, for every module with an active session. |
| `/generate` | POST | Main inference endpoint. Streams a response over SSE. |
| `/debug/router` | GET | Dry-run of keyword-only routing — shows which module a prompt *would* hit, without the full hybrid-score path. |
| `/debug/telemetry` | GET | Raw GPU/CPU/RAM stats. |
| `/debug/status` | GET | Active module, full trigger matrix, telemetry. |
| `/debug/toggle` | GET | Reports current debug-mode state. |

Debug routes are meant to be gated and return an auth error when disabled.

**The `/generate` flow, as designed:**
1. Resolve routing (the logic above) to get a model, its system prompt, its session history, and the cleaned message.
2. Retrieve a small number of prior memory findings scoped to *that module only*, injected into the system prompt if any exist.
3. Check the token budget — surfaced as a structured response, not a raw HTTP error, if the prompt would exceed the module's limit.
4. Stream tokens back over SSE: a module-identification event first, then token events, then a completion event.
5. After streaming, update that module's session history and write a truncated finding back into its memory slot automatically — not a separate, optional call.

## Frontend (conceptual)

A single-page UI, light-themed, meant to show: a telemetry drawer (VRAM/GPU/CPU/RAM, memory count, session-length indicator, active module), a chat column, and an active-module indicator card. The chat flow is designed to open an SSE stream, dispatch module/token events to the UI as they arrive, and update the module indicator the moment routing resolves — before generation even finishes.

## Known Limitations

An honest account of open design questions, not a polished feature list:

- **No fail-safe if the toxicity-audit models fail to load.** Because the audit is designed to run unconditionally before any module is reached, a failed model load currently has no graceful degradation path in this design — every query would be affected, not just edge cases. This is a real single-point-of-failure risk introduced by making the audit unconditional, and it isn't resolved yet.
- **A dedicated shared-state gateway component, as originally envisioned, is a simpler single-instance design in practice** — not the more elaborate cumulative-token-counting gateway sketched early on.
- **A VRAM-percentage auto-flush governor was planned but isn't part of the design yet** — telemetry reporting is there; an active control loop reacting to it is not.
- **MainCore is conceived as just another specialist module**, not a separate dedicated runtime layer, despite earlier framing suggesting otherwise.
- **An earlier design let a direct name-call route straight to the Social module before any audit ran** — meaning harmful phrasing could slip through if it wasn't on an exact instant-trigger string. Corrected so the audit gate applies before name/prefix resolution, not after.
- **A swap-event log for the frontend is planned but not wired to real events yet.**

## Philosophy

No cloud dependency, no telemetry, no external API calls. Every routing decision — semantic, audit-based, or hard-triggered — is designed to happen locally, deterministically, and to be fully inspectable by the person running it.

## AI assistant context

If you're using an AI assistant to think through something similar, share this document first — it describes the intended architecture, the routing logic, and the known open questions, which makes that conversation far more useful than starting from scratch.
