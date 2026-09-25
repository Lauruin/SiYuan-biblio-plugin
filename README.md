# SiYuan Biblio Plugin

Planning docs for a self-hosted knowledge/note-taking setup built on [SiYuan](https://github.com/siyuan-note/siyuan), extended with a Retrieval-Augmented-Generation (RAG) assistant, a self-hosted CalDAV task/calendar pipeline, and an optional game-skin (Phaser + React) view of the same notes as an explorable pixel-art library.

**Status: planning phase, no code yet.** This repo currently holds architecture and research docs written before implementation starts.

## Doc map

- [`HANDOFF.md`](HANDOFF.md) — current status and links to everything below; read this first.
- [`ueberblick.md`](ueberblick.md) — high-level overview (German).
- [`architecture-scope.md`](architecture-scope.md) — three-plane architecture (SiYuan + CalDAV data plane, intelligence service, game-skin client) and build order.
- [`backend-research.md`](backend-research.md) — why SiYuan, alternatives considered, gap analysis.
- [`scheduling-calendar.md`](scheduling-calendar.md) — todo/calendar subsystem (Radicale/CalDAV).
- [`game-ui.md`](game-ui.md) — the optional game-skin client concept.
- [`persona-debate.md`](persona-debate.md) — multi-persona critique of the plan.
- [`thoughts.md`](thoughts.md) — original wishlist/source of truth.
- [`infrastructure/networking-access.md`](infrastructure/networking-access.md) — remote access, backups, ops.

## Priorities

Quick capture of thoughts, reliable todos/calendar, and full sync take priority over the game UI — see `architecture-scope.md` for the intended build order (intelligence service → calendar/task pipeline → game UI).
