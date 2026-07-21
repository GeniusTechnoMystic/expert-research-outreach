# Session Journal — 2026-07-21: Expert Research & Outreach Infrastructure

**Session type:** Research + Planning + Documentation
**Duration:** Jul 21, 2026 (as part of multi-project session)
**Main project:** Expert Research & Outreach Infrastructure
**Related skill:** `research/expert-research-outreach-infrastructure` (created)

---

## Summary

Researched and designed an infrastructure layer to let Hermes reach beyond the digitized web — contacting human experts, scholars, and academics via email and forums, and orchestrating digitization requests for offline archive materials.

## Work Done

### Research Phase
- **Expert networks:** Surveyed GLG, AlphaSights, Guidepoint, Third Bridge, CleverX — mostly paid consulting ($500-1,500/call). CleverX has accessible API at ~$50-200/call. Guidepoint has MCP integration for transcript library.
- **Academic social networks:** ResearchGate (20M+, Q&A feature, no API), Academia.edu (80M+, weak programmatic), ORCID (10M+, full REST API)
- **Q&A platforms:** Stack Exchange (184 sites, **best automation** — full REST API with OAuth + questions.add endpoint), Reddit (PRAW, strong API), ResearchGate (Playwright needed)
- **Researcher discovery:** OpenAlex (300M+ papers, free API, 100K req/day, email in metadata ~30%), Semantic Scholar (200M+, free API, author + citation graph), ORCID (public email ~40%)
- **Email discovery:** Hunter.io (68% accuracy, free tier 25/mo), pattern guessing (first.last@domain via ROR ID reverse)
- **Digitization:** Archive.org API, HathiTrust API, university special collections web forms
- **Email server:** Stalwart (Rust, ~180MB RAM, native NixOS module) → best fit for NixOS microVM

### Created Deliverables

| Artifact | Path |
|----------|------|
| Project brief | `~/workspace/documents/Projects/Expert-Research-Outreach/Expert_Research_Outreach_Brief.md` |
| Implementation plan (74 tasks, 6 phases) | `~/workspace/documents/Projects/Expert-Research-Outreach/Implementation_Plan.md` |
| Hermes action plan | `~/.hermes/plans/2026-07-21_expert-research-outreach.md` |
| Hermes skill | `research/expert-research-outreach-infrastructure` |
| Hermes kanban project | `expert-research-outreach` (7 seeded tasks) |
| PROJECT_INDEX.md updated | Added to Tier 2 (Design & Build) + Hermes Projects table |
| PROJECT_STATUS.md updated | Added Design section with phase breakdown |

### Key Architecture Decisions
1. **Stalwart over Mailcow** — Rust, 180MB vs 2-6GB, native NixOS module
2. **Stack Exchange as primary forum channel** — full REST API with OAuth, no scraping
3. **OpenAlex + Semantic Scholar dual API** — redundancy and coverage
4. **Phase order:** Foundation (email) → Discovery → Enrichment → Outbound + Digitization
5. **$5.50/mo Hetzner CX22** sufficient for Stalwart + all scripts
6. **IP warmup critical** — rushing deliverability permanently damages domain reputation

### Next Actions (when ready)
- Phase 1: Choose domain, provision Hetzner VPS
- Phase 1.3: Deploy Stalwart on NixOS with services.stalwart
- Phase 2: Write discovery scripts for OpenAlex/Semantic Scholar/ORCID

### Kanban Tasks Created
| ID | Task | Phase |
|----|------|-------|
| t_8d88d0bc | P1.1 — Domain registration + DNS setup | 1 |
| t_43f9b93e | P1.2 — VPS provisioning + firewall + TLS | 1 |
| t_68a4cfee | P1.3 — Stalwart mail server deployment | 1 |
| t_9def218d | P1.4 — IP warmup script + delivery monitoring | 1 |
| t_1db105f2 | P2.1 — OpenAlex + Semantic Scholar + ORCID scripts | 2 |
| t_40ef14a1 | P3.1 — Email enrichment pipeline | 3 |
| t_ff5c1d80 | P4.1 — Multi-channel outbound (SE + Reddit + RG + SMTP) | 4 |
| t_90921542 | P5.1 — Digitization pipeline | 5 |
| t_b6497fb9 | P6.1 — Response handling + testing | 6 |

All 9 tasks in ready status. Total task plan: 74 across 6 phases (~26.5h estimated).
