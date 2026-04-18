# MemOS (Memory OS) — Understanding Notes

This note summarizes what MemOS is, how it is structured, and common integration patterns when adopting MemOS inside another application or agent runtime.

## What MemOS is

MemOS is a **memory operating system for LLM applications**. It provides:
- Persistent, versioned long-term memory objects (not just logs)
- Retrieval modes (vector similarity, BM25/full-text, and hybrid mixture)
- Optional graph-backed deduplication/conflict tracking and richer relationship modeling
- A configurable extraction pipeline that turns raw conversation/content into structured memory items
- Optional service mode (FastAPI) and integration protocols (including MCP)

A key design choice: MemOS treats memory as a managed system resource with lifecycle, feedback, and evolution/versioning.

## Architecture at a glance

Typical layering:
- **API layer** (FastAPI): REST endpoints for add/search/update/delete and admin operations
- **MOS core / orchestration**: manages one or more “memory cubes” and coordinates operations
- **MemCube container** (GeneralMemCube): groups memory subsystems (textual, activation, parametric, preference)
- **Pluggable backends**: LLMs, embedders, vector DBs (e.g., Qdrant/Milvus), graph DBs (e.g., Neo4j/Postgres/PolarDB)

The project is heavily **config/factory driven**: major components are instantiated via config models and `from_config()` patterns.

## Memory model (conceptual)

MemOS stores “textual memory items” with metadata and lifecycle.
Common attributes you’ll see:
- `id` (UUID)
- `memory` (content)
- `metadata` (type/tags/user/session/source)
- versioning/lifecycle fields (e.g., status, version history, evolve-to links)

This is more structured than an append-only log; updates can increment versions and retain history.

## Retrieval/search modes

MemOS supports multiple retrieval strategies:
- **FAST**: embedding similarity (vector search)
- **FINE**: BM25/full-text style search (and/or graph DB involvement)
- **MIXTURE**: hybrid retrieval plus deduplication (MMR-like behavior)

A practical takeaway for integrations: you can start with a minimal backend, then add stronger retrieval/dedup as operational complexity is acceptable.

## How MemOS is commonly used

There are generally three integration patterns:

1) **Library mode (in-process)**
- Your agent/harness imports MemOS classes and calls `add/search/update/delete`.
- Best for a single-user, local-first workflow.

2) **Service mode (API)**
- MemOS runs as a server; your harness uses a client to call it.
- Best when you want multi-user, shared memory, or a decoupled deployment.

3) **Protocol/tooling mode (MCP)**
- MemOS exposes tools; an agent runtime calls them via an MCP server.
- Best when your harness supports tool calling rather than direct imports.

## What MemOS owns (in a system)

In a larger system, MemOS is best treated as the **memory subsystem**:
- ingestion/extraction into structured memory items
- storage (local DBs or external services)
- retrieval (semantic + full-text + hybrid)
- lifecycle (update/versioning/history/evolution)
- provenance (sources/metadata)

What MemOS typically does *not* replace:
- project governance / policy (permissions, tool allowlists)
- human review gates and editorial control (unless you explicitly build it)
- repository-level “constitution” documents (PRD/protocols)

## What to be careful about when integrating

Some MemOS subsystems pull in heavier operational dependencies:
- Scheduler/async processing layers may assume Redis + a relational DB and an ORM schema.
- The API layer may assume DB-backed auth/key management.
- Graph-backed features assume you’ve provisioned and managed the graph store.

If your goal is “simple but serious” on Windows, start minimal:
- Prefer an in-process memory backend that works without external services.
- Add Qdrant (local docker) only once you need higher-quality vector retrieval.
- Treat graph DB + schedulers as “phase 2+” features.

## Suggested “minimal viable integration” (generic)

A safe first milestone for integrating MemOS into an existing agent or app:
- Keep your current human-auditable rules/policies as the source of truth.
- Use MemOS first for **retrieval acceleration** (search) rather than silently replacing your governance layer.
- Make MemOS optional behind a feature flag so the system can run without external services.
- Start with a minimal backend (local-only) and add services (Qdrant/graph/scheduler) only when you have operational confidence.

## Practical integration checklist

When wiring MemOS into another codebase, you usually need to decide:
- **Deployment**: in-process library vs service (REST) vs tool/protocol (MCP)
- **Storage location**: per-user local dir vs per-repo dir vs shared service
- **Backends**: (a) no-service local store, (b) Qdrant/Milvus, (c) graph DB for dedup/relations
- **Ingestion**: what gets extracted, when, and with what LLM
- **Safety**: what memory types are permitted (preferences? tool traces? secrets?)
- **Review**: whether updates/evolution require explicit human review
