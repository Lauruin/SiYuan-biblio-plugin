# Architecture Scope — Knowledge App

High-level overview. For topic detail, see: `backend-research.md` (why SiYuan, gap analysis, alternatives), `game-ui.md` (the game-skin client in full), `scheduling-calendar.md` (todo/calendar), `infrastructure/networking-access.md` (remote access). `HANDOFF.md` is the doc map and current status.

## Three planes

```
┌─────────────────────────────┐   ┌─────────────────────────────┐
│  Presentation plane          │   │  (same backend, two skins)   │
│  A) SiYuan native web UI     │   │  B) Game-skin client         │
│     (reused as-is, "utility  │   │     (custom, Phaser +        │
│     mode" / terminal-ish)    │   │     React overlay — see      │
│                              │   │     game-ui.md)              │
└──────────────┬───────────────┘   └───────────────┬───────────────┘
               │  REST/WebSocket (kernel API)        │
               ▼                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Data plane — SiYuan kernel (Go, self-hosted via Docker)         │
│  .sy files = source of truth · SQLite = rebuildable index        │
│  Dejavu = versioning + encrypted sync/backup                     │
│  + Radicale (CalDAV, see scheduling-calendar.md)                  │
└──────────────────────────────┬────────────────────────────────────┘
                               │  read notes / write back tags,links
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Intelligence plane — own microservice (Python/FastAPI or Node)  │
│  - embeds notes → vector index (sqlite-vec or Chroma)            │
│  - semantic search + RAG chat endpoint                           │
│  - agent personas (Socratic tutor etc.) via OpenRouter,          │
│    bring-your-own-key, cheap model default                       │
│  - minigame/quiz generation (later, reuses same RAG context)     │
└─────────────────────────────────────────────────────────────────┘
```

Why this shape: SiYuan already solved the hard, boring 80% (conflict-safe sync, mobile clients, markdown/table editing, encrypted backup via Dejavu). Radicale solves the same kind of boring-but-hard problem for scheduling (recurrence, timezones). Only two genuinely new things get built: the intelligence/RAG service and the game UI. Neither needs to touch SiYuan's Go core — everything talks to it over its existing kernel API.

## Component notes

**SiYuan kernel** — run self-hosted (Docker), reachable only over Tailscale (see `infrastructure/networking-access.md`), never public. Full reasoning for the choice in `backend-research.md`.

**Radicale** — self-hosted CalDAV/CardDAV alongside SiYuan. Full reasoning in `scheduling-calendar.md`.

**Intelligence service** — the one genuinely new piece of backend infrastructure.
- Indexing: pull note content via kernel API (`/api/query/sql`, filetree endpoints) or watch `.sy` files directly; chunk + embed (local embedding model, e.g. a small sentence-transformer, to avoid paying per-note); store in `sqlite-vec` (simplest — no extra service to run) or Chroma if a nicer client library is wanted.
- Chat/agent endpoint: takes a query + persona, retrieves top-k chunks, calls OpenRouter with a cheap model (e.g. a Gemini Flash / DeepSeek-class model) using a personal API key.
- **Open tension, flagged 2026-09-25, not yet resolved**: sending note content to OpenRouter's cloud models is in direct tension with every other decision in this project — self-hosted notes, self-hosted calendar, VPN-only access, ruling out cloud services on privacy principle elsewhere. Two real options, categorically different, not a spectrum: (a) a **local model** (e.g. via Ollama on own hardware) — genuinely private, no third party ever sees note content, but weaker capability and needs real GPU/CPU headroom; (b) a **GDPR/DSGVO-conforming hosted provider** — legally constrained in how it handles data, but data still leaves the house and a third party still sees plaintext. (a) and (b) are not equivalent privacy levels — GDPR compliance is a regulatory property, not a non-disclosure guarantee. Decide before building the chat/agent endpoint, not after.
- Agents are just system prompts + tool access to the kernel API (create note, add backlink, tag) — no need for a heavy agent framework; a hand-rolled tool-calling loop is enough at this scope and keeps cost/behavior under control.
- Plain Python/TS, no exotic dependencies — this is the piece to build iteratively, session by session.

**Game-skin client** — separate web app. Full concept, perspective, room/door model, texture packs, dynamic tilemap, and future game modes in `game-ui.md`.

## Portability discipline (not an abstraction layer) — 2026-09-25

Considered building a formal pluggable-backend abstraction (SiYuan/Trilium/Obsidian interchangeable) — rejected as premature (no second backend exists to prove the abstraction against, and it's exactly the kind of speculative generalization the persona debate's Scope Cutter would flag). Instead, cheap discipline now that pays off if a port is ever wanted:

- **Isolate every SiYuan API call behind one clearly-bounded module** in both the intelligence service and the game UI, rather than scattering calls throughout. Costs almost nothing now; means a future port touches one file, not the whole codebase.
- **Prefer operations that exist across note apps** (tree walk, attributes, full-text search, content read/write) over SiYuan-specific mechanics where there's a real choice (e.g. SiYuan's raw SQL query endpoint has no equivalent elsewhere — use it, but don't build core logic that *requires* it if a tree-walk or search call would do).
- **Trilium is a realistic future port target** — its ETAPI has the same shape as SiYuan's kernel API (note/block tree, attachable attributes, external REST API, full-text/query search); porting would be translation work, not a redesign.
- **Obsidian is not** — no official server API at all, just markdown files + YAML frontmatter on disk, third-party-plugin-dependent for any HTTP access, no native block-ID or attribute-view system. Supporting it would be a genuinely separate integration (direct file read/write), not a variant of the same adapter. Name this honestly if it ever comes up rather than assuming the same discipline covers it.

## Suggested build order

1. Self-host SiYuan, live with it in plain form for a bit to confirm the backend choice holds up daily. (In progress as of 2026-09-22.)
2. Build the intelligence service (indexing + RAG chat + one agent persona) against SiYuan's API, tested from a bare page or SiYuan's own UI — no game UI yet.
3. Get the calendar/task pipeline (Radicale + client-side module) working and tested in plain form — verify the boring-but-correct thing works before skinning it.
4. Only once 1–3 feel solid, start the game-skin client, mapping room/shelf/book/quest-board to real folder/tag/task data from day one rather than mocking it.
