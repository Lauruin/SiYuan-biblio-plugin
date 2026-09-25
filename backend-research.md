# Backend Research — Knowledge App

Which note-taking backend to build on, and why. See `HANDOFF.md` for the doc map and current status.

## Decision: SiYuan

**Backend: SiYuan** (self-hosted, Docker), chosen over Logseq/AFFiNE/Anytype/Trilium/Reor after comparing against the wishlist in `thoughts.md`. Short version: SiYuan is the only candidate with native Android/iOS apps (real `gomobile` builds, not a broken wrapper like AFFiNE's), a Go kernel (mainstream, legible if extension past the plugin sandbox is ever needed), block-based linking, database views, built-in encrypted sync (Dejavu), and an OpenAI-compatible AI config slot (OpenRouter plugs in directly for basic chat).

**Plugins are TypeScript/JS regardless of core language** — SiYuan's core is Go, but plugin dev (via the `siyuan` npm package / `petal` API declarations) is plain TS, same as Logseq's. No Go needed unless the kernel itself is ever patched.

## Gap analysis — what SiYuan does NOT cover (confirmed against thoughts.md)

Needs custom build:
- **RAG / vector DB over notes** — not native. Plan: separate "intelligence service" (Python or Node), embeds notes (local embedding model or API), stores in `sqlite-vec` or Chroma, exposes a chat/search endpoint. SiYuan's built-in AI config covers plain OpenRouter chat but not retrieval over the vault.
- **Agent personas (Socratic tutor etc.)** — build as system prompts + a hand-rolled tool-calling loop in the intelligence service (no heavy agent framework needed at this scale).
- **Canvas/whiteboard notes** — SiYuan's own whiteboard feature ("Card Whiteboard / Maze", [issue #2024](https://github.com/siyuan-note/siyuan/issues/2024)) has been open since 2021 and was still explicitly "no resources this year" as of the last dev comment (2024) — treat as indefinitely backlogged, do NOT wait for it. Plan: own "corkboard" room in the game UI (pin+string over embedded note cards) — see `game-ui.md`.
- **iPad/browser drawing (ink)** — no native drawing tool either (separate gap from the whiteboard one). Plan: embed an existing FOSS canvas lib (tldraw/Excalidraw-style) rather than building one.
- **Automatic keyword/context auto-linking** (nice-to-have) — not native; cheap to add later once the intelligence service already has embeddings (similarity-based link suggestions).
- **Multi-user note sharing** (nice-to-have) — SiYuan sync is single-user-across-own-devices; no real collaborative sharing. Only relevant if specific notes ever need sharing with other people.
- **"Agentic management" (chat to add a feature/plugin)** (nice-to-have) — not an app feature to build; this is just the dev workflow already happening in this project. Dropped from scope as a product feature.

Everything else on the must-have list (markdown/tables, folders, unfiled notes via notebook-root docs, manual linking, self-hosted sync, encrypted backup, Android app) is already covered natively by SiYuan — confirmed, not just assumed.

## Other apps considered and ruled out (2026-09-23)

User's own hard requirement, restated plainly: fully self-hosted/local, nothing kept "remotely private" on someone else's cloud. This rules out most of the following on its own, before even checking feature fit.

- **Heptabase** — closed-source SaaS, no self-host, no API, $7-18/mo, no permanent free tier. Nice card-on-whiteboard UX; reviewers themselves point to Logseq whiteboards as the FOSS equivalent.
- **Capacities** — hosted-only, proprietary data format, freemium ($10/mo Pro). Their own comparison page tells privacy-focused users to use Anytype instead.
- **Tana** — worst fit: cloud-only, **no offline mode, no local storage at all**. Hard disqualifier regardless of price. Note: "Tana" split in 2026 into an outliner product and an unrelated AI meeting-assistant product sharing the brand name — don't conflate them if researching further.
- **Hillnote** — closest in spirit (local-first, plain-markdown storage, offline local AI via Ollama) but the app itself is closed-source; the GitHub repo is a docs/showcase page, not the real source. Sync/frontier-AI are paid cloud add-ons. Useful as a UX reference only (local files + optional cloud AI hand-off pattern), not adoptable.
- **Milanote** — cloud-only, closed-source, no self-host, no AI. AFFiNE remains the FOSS analog for this use case, same self-host-complexity/polish tradeoff.
- **Griply** — not actually a notes/PKM competitor — it's a goals/habits/task app. Closed-source, no Android app (PWA only), no API/export. Not adoptable — see `scheduling-calendar.md` for the actual todo/task plan.
- **hamsterbase** — two unrelated projects share the name: "HamsterBase" (web-archive/bookmarking tool, SDK-only open source, not fully OSS) vs. **HamsterBase Tasks** (genuinely open source, local-first, E2E-encrypted task app — the relevant one, kept as reference prior art only).

## Trilium/TriliumNext reconsidered (2026-09-24)

Original assessment called Trilium "stable, maintenance mode, no canvas, no AI/RAG." That's now outdated — worth flagging honestly since it changes the comparison:

- **TriliumNext took over active development** after original author zadam stepped back, and has since reclaimed the "Trilium Notes" name — this is now the actively developed lineage, not a stale fork. 100% TypeScript (server + client) as of recent releases.
- **Has built-in canvas** (via Excalidraw) — something SiYuan still lacks (Maze/whiteboard indefinitely backlogged, see above). Also has mind maps, spreadsheets, geo maps built in.
- **Has opt-in AI**: BYO-key chat (OpenAI/Anthropic/Google/self-hosted OpenAI-compatible) that "draws on your notes," plus MCP exposure for external assistants. Unclear whether this is true embedding-based RAG retrieval or simpler context-stuffing — treat as a head start, not a replacement for building real semantic search if the vault gets large.
- **Scripting**: JS code notes + a real Script API (frontend + backend) + a separate stable ETAPI for external integrations — comparable to or better-documented than SiYuan's plugin situation.
- **License**: AGPL-3.0, same as SiYuan — same commercial/self-hosting analysis applies.
- **Where it's worse than SiYuan, and why SiYuan was still picked**: mobile is the weak point. No official native app — relies on **TriliumDroid** (community Android client, sync version must match server exactly, fragile) and **Trinote** (community iOS client), vs. SiYuan's official first-party Android/iOS/HarmonyOS builds via `gomobile`. This was the exact dealbreaker that ruled out AFFiNE, and it applies to Trilium too, just less severely (a working community client exists, unlike AFFiNE's broken one).

**Net**: genuinely closer than initially assessed — closes two of SiYuan's real gaps (canvas, some AI) — but still loses on the one requirement that mattered most (a real, official, first-party Android app). Not a reason to switch given the SiYuan trial is already underway; worth trying self-hosted in parallel for a few days if there's ever doubt, since standing both up is cheap compared to committing code to one.

## Plugin ecosystem — where to look, and a reality check

**Where to find plugins:**
- **In-app**: Settings → Marketplace (the live, authoritative source — actual install/download counts, not visible from outside the app).
- **[siyuan-note/bazaar](https://github.com/siyuan-note/bazaar)** `plugins.txt` — the raw list that powers the in-app marketplace; browsable on GitHub without opening the app.
- **[siyuan-note/awesome-siyuan](https://github.com/siyuan-note/awesome-siyuan)** — official community-curated hub for plugins, themes, widgets, tutorials.
- **GitHub topic [`siyuan-plugin`](https://github.com/topics/siyuan-plugin)** — sortable by stars, closest thing to a popularity signal outside the app.

**Reality check**: the ecosystem is small compared to Obsidian's — top community plugins sit in the 15-45 GitHub-star range, not thousands, and a fair amount of documentation/UI is Chinese-first (matches the userbase/company background). Don't expect Obsidian-scale plugin depth; do expect the basics to be covered.

**Named plugins worth knowing about:**
- **[Better Sync](https://github.com/DD3Boh/better-sync-siyuan)** — free, P2P sync between your own SiYuan instances, no Pro license/subscription needed. Directly relevant to the self-hosting plan as an alternative to S3/WebDAV. Actively developed but the author's own README warns it can still cause sync conflicts/data loss — back up before relying on it.
- **sy-query-view** — turns saved queries into a dashboard view (useful once there are enough notes to want an overview).
- **Leaf Nest** — broader "full workflow" note-taking plugin bundle.
- **Spaced Repetition System** — flashcard/SRS plugin, complements SiYuan's own built-in flashcard feature.
- **Web Clipper** (Chrome/Edge extension) — clip web pages straight into SiYuan.
- Smaller utility plugins: Format Painter, Bookmark plugin, Snippets manager, Document Navigation — quality-of-life, not architecture-relevant.
- **[siyuan-plugin-caldav-sync](https://github.com/bonebearHsu/siyuan-plugin-caldav-sync)** — see `scheduling-calendar.md`, the relevant one for the todo/calendar integration.

None of the general-QoL plugins are required for the RAG/agent/game-UI plan — optional additions to try once living in the app day-to-day. Better Sync is the one worth testing early given the self-host-only decision.

## Company/licensing note (context, not a decision point)

SiYuan is built by B3log (Yunnan Liandi Technology Co., Ltd., China), AGPL-3.0, small community-driven studio (also makes Vditor, Solo). Core app fully self-hostable with no feature gate; their paid tiers only add their own cloud sync. Self-hosting means never depending on their cloud, and AGPL means forkable if the project stalls.

**Commercial licensing implications** (asked out of curiosity, not a current decision): Phaser is MIT (fully commercial-safe). SiYuan's AGPL-3.0 only imposes obligations if its own source is modified and run/distributed as a network service — building separate code that talks to its unmodified API (the plan here) carries no license obligation on that separate code. Art asset licenses vary per-pack — see `game-ui.md` for which packs are CC0 (no restriction) vs. share-alike.
