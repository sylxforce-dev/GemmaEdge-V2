GemmaEdgeV2

A fully offline LLM runtime optimized for 8GB VRAM hardware, designed for multi-agent orchestration without memory fragmentation.

Core Capabilities
Memory Management: Atomic load/unload cycles for single-model isolation.
Deterministic Routing: MishaSelector semantic pipeline using CPU-bound embedding backends.
Storage: Agent-specific RAM-based vector stores for zero semantic bleed.

Architecture overview only — implementation details, routing thresholds, and trigger logic are kept private.

Where to look
BLUEPRINT.md — the full design writeup: routing logic, VRAM management, module structure, config, known limitations. Start here for the "why."
LOGS.md — real, unedited terminal output from actual test runs, proving the design in BLUEPRINT.md out in practice. Start here for the "does it actually work."
