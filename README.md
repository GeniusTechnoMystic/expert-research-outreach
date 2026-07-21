# Expert Research & Outreach Infrastructure

**Purpose:** Infrastructure layer that lets Hermes reach beyond the digitized web — into human expertise, offline archives, and institutional collections. Self-hosted email, researcher discovery APIs, multi-channel forum Q&A, and automated digitization requests.

**Status:** Design / Not Started
**Created:** 2026-07-21
**GitHub:** https://github.com/GeniusTechnoMystic/expert-research-outreach
**Kanban slug:** `expert-research-outreach`
**Skill:** `research/expert-research-outreach-infrastructure`

## Contents

| File | Description |
|------|-------------|
| `Expert_Research_Outreach_Brief.md` | Vision, architecture, 6-tier channel analysis, Stalwart config, cost estimates, 8-phase breakdown |
| `Implementation_Plan.md` | 90 tasks across 8 phases, dependency tables, risk register, queue discipline, feedback loop, circuit breaker, TOS compliance |
| `session-journal-2026-07-21.md` | Full session log: research findings, decisions, artifacts created, kanban seeding |

## Architecture (v2.0)

```
Discovery → Enrichment → Review Gate → Priority Queue → Outbound 4A (API)
                                                      → Outbound 4B (browser)
                                Digitization (parallel)
                                Feedback Loop ← replies ←
```

## Key Design Decisions

- **Stalwart > Mailcow** — Rust single binary, ~180MB RAM, native NixOS module
- **Stack Exchange > other forums** — full REST API with OAuth, no scraping
- **OpenAlex + Semantic Scholar dual** — redundancy, free APIs
- **Circuit breaker** — hard stop on bounce >3% or any spam complaint
- **Human review gate** — auto-promotes after 5 clean batches
- **Feedback loop** — replies feed back into discovery ranking

## Related Projects

- **PC Design Agent** — `~/workspace/documents/Projects/PC-Design-Agent/` (sibling project)
- **PROJECT_INDEX.md** — `~/workspace/PROJECT_INDEX.md` (master catalog)
- **PROJECT_STATUS.md** — `~/workspace/PROJECT_STATUS.md` (live dashboard)