# Scheduling & Calendar — Knowledge App

Todo/task/calendar plan, in full. See `HANDOFF.md` for the doc map and current status; see `game-ui.md` for the "quest board" room that visualizes this data in-game.

## Why this exists as its own subsystem

**Todo/task feature — validated as a real gap**, not solved by any of the note-app alternatives considered in `backend-research.md`. Don't build recurrence/timezone/scheduling logic from scratch — that's a solved-protocol problem (same "adopt the hardened boring 80%" reasoning as picking SiYuan itself over building a notes app).

## Decision: self-hosted Radicale (CalDAV)

Adopt **[Radicale](https://radicale.org/)**, a small, actively maintained, GPL-3.0 self-hosted CalDAV (calendars + VTODO tasks) + CardDAV server, pure Python, file-based (no DB), trivial to run in Docker alongside SiYuan. This inherits RRULE recurrence, timezone handling, and real task objects (VTODO — due dates, priority, completion) for free, plus standard-CalDAV interop (Apple Calendar, Thunderbird, DAVx⁵ on Android) with zero extra UI work if ever wanted.

- **Cross-referencing with SiYuan**: a custom property on each VTODO points back to the SiYuan block ID it originated from; a mirrored `custom-task-uid` attribute on that block points to the task. The quest-board room (see `game-ui.md`) joins both without either system owning the other's data.
- **Client**: the mature Python `caldav` library talks to Radicale from the same backend service already being built for RAG — no protocol-level work needed.

## Architecture: client-side module, not a plugin or new service

The calendar bridge is **not** a SiYuan plugin (plugins only run while SiYuan's own app is open — wrong shape for something that must work independent of that, and would silo the data from the game UI) **and not a dedicated backend service either** (unlike RAG, CalDAV needs no server-side compute/API-key custody — the "client" role is exactly what the game UI can play directly). Decision: a **client-side CalDAV module inside the game UI** (e.g. via the `tsdav` JS library), talking directly to Radicale and to SiYuan's kernel API for cross-referencing, with local caching (IndexedDB) for offline tolerance (see `infrastructure/networking-access.md` for the connectivity constraint this satisfies). The only genuinely separate backend service in the whole plan remains the intelligence/RAG service.

## Conflict resolution — explicit decision, 2026-09-25

Previously unaddressed: what happens if a task is edited on the phone while offline, and the same task is edited from the SiYuan plugin (or another CalDAV client) before the phone ever reconnects? This is a real gap in the local-cache-first architecture (see `infrastructure/networking-access.md`), not something SiYuan or Radicale silently solves — CalDAV/iCalendar's native conflict handling is **last-write-wins at the object level** (whichever client's sync completes last overwrites), no smart merge. (Note: this was nearly missed by assuming "SiYuan probably has a good solution" — it doesn't own this collision path at all, and the project's own research on Better Sync already documented that even SiYuan's own sync "can still cause sync conflicts/data loss.")

**Decision: accept last-write-wins as the working policy.** For a single-user app, the actual rate of two genuinely simultaneous edits to the same task within one offline window is low — building real conflict merging isn't proportionate to the risk. This is an explicit, accepted risk, not an oversight. Revisit only if it actually causes a real data-loss incident in practice.

## Google Calendar — migrating away, not integrating

**Google/Outlook calendar integration would be a separate, larger ask** — not something CalDAV bundles in automatically. Google's CalDAV interface is pull-only (no bidirectional server-to-server sync), OAuth2-gated (token refresh/expiry to manage), and **does not support tasks (VTODO) at all**, only events. Apple/Nextcloud calendars are proper two-way CalDAV natively and would just work through the same code.

**Resolved 2026-09-24**: not needed — the user is switching away from Google Calendar entirely (not keeping it connected). **Decision: self-hosted Radicale, one-time migration from Google Calendar** (export via Google Takeout/ICS, import into Radicale) — no ongoing Google integration needed.

## Proton Calendar — checked and ruled out as the backend

User has Proton Premium and asked whether Proton Calendar could serve as the backend instead of self-hosting. Checked — **Proton Calendar has no CalDAV support at all**, architectural (their zero-access encryption model is incompatible with a protocol that requires server-side read access), only one-way ICS import/export and read-only "subscribe" feeds. Unofficial reverse-engineered bridges exist (Protoxide, carbonate) but depend on unofficial Proton API access that could break anytime — not a good foundation given the user explicitly wants "good implementation" for todos/calendar specifically. Proton could still *subscribe* read-only to a Radicale ICS feed later if wanted, since that one-way path is something Proton does support — but it's not the backend.

## Android FOSS client stack (2026-09-24)

**DAVx⁵** (F-Droid, GPLv3, actively maintained) as the sync engine — it doesn't render its own UI, it syncs Radicale's data into Android's system calendar/contacts storage. Pair with **Etar** or **Fossify Calendar** for events, and **Tasks.org** specifically for tasks (VTODO) — not every FOSS calendar app supports CalDAV tasks, only Tasks.org/OpenTasks/jtx Board do, so events and tasks need separate apps in the fully-FOSS stack.

**jtx Board considered**: Android app combining journals/notes/tasks (VJOURNAL+VTODO) with linking between them — closer in spirit to "notes and tasks connected" than Tasks.org, F-Droid `.ose` build is genuinely open source, actively maintained. Three caveats before treating it as a replacement for the DAVx⁵/Etar/Tasks.org stack:
1. **Still requires DAVx⁵ underneath** — it's a front-end, not its own sync engine.
2. **Calendar/event viewing is deliberately basic by the developer's own admission** ("I don't want to do a fully-fledged calendar app... show entries to allow linking") — not a real calendar view, still want Etar/Fossify alongside it for that.
3. **Multiple user reports of unreliable CalDAV VTODO sync** (completed tasks reappearing as open, stale reminders) with some servers — a real flag given the stated reliability priority; test carefully against Radicale specifically before trusting it, don't assume other users' server pairings predict this one.

## SiYuan-side integration — revised 2026-09-25, avoid the native-plugin dependency

**siyuan-plugin-caldav-sync, checked more precisely**: worse risk profile than "small hobby project" suggested — it's **brand new** (created September 2026), 3 GitHub stars, effectively single-maintainer (two GitHub accounts, identical code — almost certainly one person), and still shipping fixes for basic bugs days apart (one release fixed silently reconnecting to the stale server address after a config change). Zero track record. Not a good foundation for the reliability-critical path.

**Better alternative found: embed a mature standalone web CalDAV client, don't depend on a native SiYuan plugin at all.** SiYuan natively supports iframe embeds (paste an `<iframe>` into a note, no plugin) and has an official "Widget" system (embeddable web apps via its Community Bazaar) — so a calendar view can live inside SiYuan's UI without any custom plugin touching SiYuan's own plugin sandbox or data.

- **[WebCalDav](https://github.com/andowero/WebCalDav)** — modern, actively maintained, single-Docker-container CalDAV web client, explicitly built to fill the gap left by two now-abandoned projects (AgenDAV, InfCloud). Ships its own **MCP server**, letting an AI assistant list/create/edit/complete tasks and events via natural language — directly useful for the intelligence service's tool-calling, essentially for free. **First choice.**
- **AgenDAV** — fallback if WebCalDav proves shaky. Genuinely battle-tested (~6 years, 26 contributors, 2,200+ commits) but now maintenance-mode only (stability/compat fixes, no new features) — more boring, more proven.
- **InfCloud** — checked and ruled out: last release 2015, effectively abandoned.

Plan: embed WebCalDav (iframe or Widget) inside SiYuan for the "boring UI" calendar view, decoupled entirely from SiYuan's plugin sandbox — isolates any WebCalDav instability from SiYuan itself. Combine with the game UI's own client-side CalDAV module (already decided above) — both talk to the same Radicale backend, neither depends on a two-week-old plugin.

## Priority context

User's stated top priorities: quick capture, and todos/calendar specifically needing good/reliable implementation — sync reliability matters more than game-UI polish. Mobile app is "nice, not a dealbreaker" (softens, doesn't reverse, the Android requirement from `backend-research.md`). **Build order adjustment**: after the intelligence service, get the calendar/task pipeline working and tested in plain form (bare page or a standard CalDAV client against Radicale) *before* building the quest-board room's visuals — verify the boring-but-correct thing works before skinning it. Also worth confirming during the SiYuan trial whether its native quick-capture flow already satisfies "put down my thoughts quickly," since that's likely already covered without any custom work.

## Open questions

- Whether siyuan-plugin-caldav-sync supports note→calendar (not just calendar→note) — untestable until servers are set up.
- jtx Board / siyuan-plugin-caldav-sync reliability against Radicale specifically — needs real testing once servers exist.
