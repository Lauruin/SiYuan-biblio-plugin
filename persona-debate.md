# Persona Debate — Knowledge App

Six-agent critical debate on the project plan, run 2026-09-25. Personas: The Scope Cutter, The Reliability Engineer, The Privacy Auditor, The Game Designer, The Future Maintainer, The Daily-Use Skeptic. Each read the current docs independently, staked a position, then rebutted each other; a final synthesis pass (with its own independent GitHub verification) compiled the result below. See `HANDOFF.md` for the doc map this refers to.

## Verification: is there really no alternative to two-author hobby plugins?

Short answer: **partial alternatives exist, but the debate slightly oversold how clean the "safe" pick is, and missed one new risk signal.**

| Repo | Stars | Forks | Contributors | Age | Last push (today = 2026-09-25) | Notes |
|---|---|---|---|---|---|---|
| `bonebearHsu/siyuan-plugin-caldav-sync` | 4 | 0 | 2 (46+2 commits) | 13 days | today | Confirmed: brand-new, effectively single-author. |
| `andowero/WebCalDav` | 4 | 1 | 1 (62 commits) | 5 months | 7 days ago | Single-maintainer but actively pushed weekly. Ships a real MCP server — **off by default, opt-in per token**; its own docs warn "an API token can decrypt the user's CalDAV credentials." No persona had checked this — lowers the severity of the earlier privacy flag somewhat, but raises the stakes if it's ever turned on. |
| `DD3Boh/better-sync-siyuan` | 122 | 8 | 1 (421 commits) | 17 months | **2026-04-29 — ~5 months stale** | New finding, nobody in the debate checked push recency: this looks currently dormant, not just bus-factor-risky. |
| `agendav/agendav` | 828 | 128 | 24 | 15 years | 12 days ago (dependency-bump PR only) | Real feature/bug work mostly stopped around July 2026; a Sept 19 bug report is still open. The debate's framing of AgenDAV as clearly-safer is **directionally right but overstated** — genuinely maintenance-mode, not thriving. |

**Structural finding nobody caught**: Radicale has no web UI of its own (file-based, admin-only) — some third-party web client is unavoidable for a browser-embedded calendar view, full stop. The one path that sidesteps the whole hobby-plugin question: the **phone-native stack (DAVx⁵ + Tasks.org/Etar)** already chosen in `scheduling-calendar.md` — mature, multi-contributor FOSS, zero bus-factor problem. That's the thing actually opened every morning; the desktop web-client choice (WebCalDav vs. AgenDAV) barely matters by comparison.

## Where all six agreed

1. **WebCalDav is a single-author, ~5-month-old project with the same risk profile as `siyuan-plugin-caldav-sync`** — the "safer replacement" relocated the bus-factor risk rather than removing it.
2. **There is no disaster-recovery/backup story for the single home machine**, and "confirm physical hosting target" is still an open question, not a blocking prerequisite. Called the single most consequential, unrecoverable gap in the whole plan.
3. **The two stated top priorities (fast capture, reliable calendar) are the least validated and least built parts of the plan, while the game world is the most fully specced.** Effort went where the docs were fun to write, not where the priority was stated to be.
4. **Cut the metroidvania combat mode and the in-game pixel editor outright, not "defer."** No persona defended keeping either; neither shares a code path with capture or todos, and the pixel editor solves a contributor-distribution problem that doesn't exist for a single-user project.
5. **The corruption mechanic ("strongest idea" per `game-ui.md`) is mechanically weak** — no tell, no fail state, no cost for missing it, resolved by a menu click.
6. **The auto-growing tilemap / texture-pack-swap abstraction is over-built relative to the actual ask** — `thoughts.md` wanted manual shelf-sorting; nobody asked for an auto-placement/bin-packing system.
7. **The OpenRouter/cloud-model privacy decision is left open in one doc but silently inherited by three later docs** (Socratic NPC, receptionist search, Phase 3 corruption/quiz generation, WebCalDav's MCP tool-calling) without ever being re-flagged at the point of use.

## Real unresolved tensions (no obvious right answer)

- **Should any game-skin work happen before capture/calendar are proven, or is a tiny vertical slice worth building now to test "is this actually fun" at all?** Scope Cutter/Daily-Use Skeptic: freeze all Phase 3 design until the boring parts survive real weeks. Game Designer: if the visual layer stays perpetually gated, the project's one actual differentiator (quest-board vs. plain checkbox) never gets tested — build one tiny, manually-placed slice specifically to falsify or confirm the hypothesis.
- **Is the "no writes to real notes" rule + cosmetic-only currency a strength or a flaw?** Reliability Engineer: one of the few genuinely good decisions, caps blast radius. Game Designer: the combined effect drains every mechanic of stakes, so "is it fun" collapses to "is walking around pleasant" — a much lower bar than the doc's own Hollow Knight comparisons imply.
- **Layout-data storage: sidecar, co-located attributes, or don't build the feature at all?** Three-way split between Reliability Engineer (sidecar), Scope Cutter (don't build it yet), and Future Maintainer (a sidecar is itself a second undocumented store that can silently go stale) — unreconciled.
- **Is WebCalDav's risk actually comparable to `siyuan-plugin-caldav-sync`'s?** Same star/contributor/age tier, but Future Maintainer's blast-radius point holds structurally: Radicale stores plain ICS/VTODO files, so WebCalDav dying just means swapping the iframe, zero data migration — a materially smaller failure than a SiYuan-sandboxed plugin with block write access.
- **Is AgenDAV actually safer than WebCalDav, or did the debate over-credit it?** "Older with more contributors" and "actually being maintained right now" point in different directions here (see table above) — no clean answer.

## Concrete recommendations (ranked by convergence + how load-bearing)

1. **Write a backup/disaster-recovery procedure; promote "confirm physical hosting target" to a blocking checklist item before real data goes in.** (Future Maintainer, Reliability Engineer, Scope Cutter, Daily-Use Skeptic)
2. **Don't treat WebCalDav-vs-AgenDAV as a priority decision at all — validate the phone-native path (DAVx⁵ + Tasks.org/Etar) first; make any desktop embedded web view optional and skippable.** (Daily-Use Skeptic, backed by the verification above)
3. **Cut the metroidvania mode and in-game pixel editor from the roadmap entirely** — someday/maybe list, not a numbered Phase 3.
4. **Ship exactly one hardcoded texture pack with pure manual placement; drop auto-growing tilemap/elevator bin-packing/manifest-swap until the static world has been lived with for weeks.**
5. **Resolve the OpenRouter-vs-local-model decision once, in privacy terms, and propagate it by reference into every downstream doc** instead of letting each silently assume an answer.
6. **If WebCalDav's MCP server is ever enabled, treat it as a high-sensitivity toggle** — its own docs confirm a token can decrypt CalDAV credentials, and it's off by default for a reason nobody had verified before now.
7. **Add a visible signal for last-write-wins conflicts** (even one log line per detected overwrite) — a silently-reverted task edit shouldn't read as "a task I forgot."
8. **Redesign or demote the corruption mechanic before ever prototyping it** — give it a real tell and a miss/catch cost, or drop it beneath the chase/catch/quiz mechanic (the only Phase 3 idea with an actual game loop).
9. **Add a short "portability/bus factor" note per datastore in `HANDOFF.md`** — raw format, survives-the-tool-dying, and where the real re-keying risk lives (Better Sync's now-stale push history and SiYuan's own block format — not the calendar-UI skin, which everyone over-focused on).
10. **Split the quest-board's data half from its presentation half explicitly** — but judge the data half's worth by actual reach-for-it speed against the phone widget, not architectural purity alone.

## What nobody challenged

- **SiYuan itself as the backend.** Every persona treated this as settled; only the Future Maintainer glancingly noted its block format is a portability lock-in with no stated export/fork plan, and SiYuan is a single company's (B3log) product. Got a pass because it was outside every lens's aperture, not because it's been stress-tested.
- **Tailscale as the network fabric.** Only the Privacy Auditor named Tailscale Inc.'s coordination/DERP role as an unexamined trust exception, as a low-severity aside. Nobody stress-tested Tailscale's own reliability (outages, account lockout) the way the CalDAV plugins got stress-tested.
- **The biggest one: whether gamifying a note app actually makes someone more likely to use it, independent of implementation.** The Game Designer came closest, questioning whether the walkable-world novelty survives daily use — but even that critique operates *inside* the premise that some game layer is worth building. No persona was assigned a "does this solve the actual human problem" lens, and it shows: six technical/design/privacy/reliability critiques of the plan, zero critiques of whether the plan's central bet — that a game skin fixes "I forget to capture thoughts and miss calendar items" — is sound at all.
