# Expert Research & Outreach Implementation Plan

**Status:** Design / Not Started
**Created:** 2026-07-21
**Project path:** `~/workspace/documents/Projects/Expert-Research-Outreach/`
**Related skill:** `research/expert-research-outreach-infrastructure`
**Kanban slug:** `expert-research-outreach`

---

## Phase 1 — Foundation: Email Infrastructure

### Domain & DNS Setup
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 1.1 | Register domain (e.g., `yourresearch.org`) | — | 15m | Registrar account + domain |
| 1.2 | Point domain nameservers to VPS DNS | 1.1 | 10m | NS records set |
| 1.3 | Set MX, A, SPF, DKIM, DMARC, MTA-STS records | 1.2 | 20m | DNS zone file |
| 1.4 | Set PTR/rdns record at VPS provider | 1.3 | 10m | Reverse DNS match |

### VPS Provisioning
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 1.5 | Provision Hetzner CX22 (or other, port 25 unblocked) | — | 15m | VPS running NixOS |
| 1.6 | Configure firewall (port 25, 465, 587, 993, 8080) | 1.5 | 10m | iptables/nftables rules |
| 1.7 | Set up ACME TLS (Let's Encrypt for mail subdomain) | 1.3 | 10m | SSL certs for mail.yourdomain.com |

### Stalwart Deployment
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 1.8 | Write Stalwart NixOS module config | 1.5 | 30m | services.stalwart stanza |
| 1.9 | Generate DKIM keys, configure DNS signing | 1.3, 1.8 | 15m | DKIM TXT record |
| 1.10 | Create mail accounts (research, postmaster, abuse, digitization, forum aliases) | 1.8 | 10m | Working mailboxes |
| 1.11 | Verify deliverability (mail-tester.com, send test to Gmail/Outlook) | 1.10 | 20m | 10/10 score |
| 1.12 | Write IP warmup script (progressive volume over 8 weeks) | 1.11 | 30m | warmup.py |

**Phase 1 Gate:** Sending and receiving email from research@yourdomain.com with 10/10 mail-tester score

---

## Phase 2 — Discovery Pipeline

### OpenAlex Integration
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 2.1 | Register OpenAlex API key (free tier, 100K req/day) | — | 5m | API key |
| 2.2 | Write OpenAlex author search by topic/concept | 2.1 | 45m | `openalex_discovery.py` |
| 2.3 | Add author filters: works_count, cited_by_count, 2yr_mean_citedness, relevance_score | 2.2 | 20m | Ranked candidate output |
| 2.4 | Extract emails from `authorships.author.email` field (where available) | 2.2 | 15m | Email-annotated candidates |
| 2.5 | Map institution→domain via ROR ID (`ror.org/{id}` → institutional email format) | 2.2 | 30m | `ror_to_domain.py` mapping |

### Semantic Scholar Integration
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 2.6 | Register Semantic Scholar API key (free) | — | 5m | API key |
| 2.7 | Write Semantic Scholar paper search by keyword | 2.6 | 30m | `semanticscholar_discovery.py` |
| 2.8 | Extract author details from paper results (name, affil, paper count) | 2.7 | 20m | Author candidate list |
| 2.9 | Cross-reference OpenAlex + Semantic Scholar results (dedup by name+affil ORCID) | 2.5, 2.8 | 20m | Merged candidate list |

### ORCID Lookup
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 2.10 | Write ORCID API query (no key needed for public data) | — | 30m | `orcid_lookup.py` |
| 2.11 | Extract public email from ORCID records | 2.10 | 15m | Verified emails |
| 2.12 | Cross-reference ORCID→OpenAlex ID to enrich candidate profiles | 2.11, 2.9 | 15m | Enriched merged list |

### Discovery Cron
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 2.13 | Write unified discovery script: topic → merged candidate list JSON | 2.9, 2.12 | 45m | `expert_discovery.py` |
| 2.14 | Register cron job: `expert-discovery` every 6h | 2.13 | 10m | Cron job running |
| 2.15 | Store results in `~/.hermes/data/research_candidates/{run_id}/` | 2.14 | 10m | Structured output dir |

**Phase 2 Gate:** `expert_discovery("LLM reasoning architecture")` returns ranked list of researchers with emails

---

## Phase 3 — Contact Enrichment

### Email Pattern Guessing
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 3.1 | Build email pattern database (common academic formats: first.last, lastf, flast, first_last@domain) | — | 30m | Pattern templates |
| 3.2 | Write pattern guesser: name + institution domain → candidate emails | 3.1 | 30m | `email_pattern_guesser.py` |
| 3.3 | Implement SMTP/MX verification for guessed emails (HELO check, no delivery) | 3.2 | 45m | Verified/non-verified flag |

### Hunter.io Integration
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 3.4 | Register Hunter.io account + obtain API key (free tier: 25 checks/mo) | — | 10m | API key |
| 3.5 | Write Hunter.io email finder: domain + name → verified email | 3.4 | 30m | `hunter_verify.py` |
| 3.6 | Write Hunter.io email verifier: check single address bounce risk | 3.4 | 20m | Verification status |

### Enrichment Pipeline
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 3.7 | Build priority queue: existing metadata email > ORCID > Hunter.io > pattern guess > SMTP verify | 3.3, 3.5, 3.6 | 30m | `email_priority_queue.py` |
| 3.8 | Write enrichment script: read discovery candidates → output enriched with contact info | 3.7 | 30m | `expert_enrich.py` |
| 3.9 | Register cron job: `expert-enrich` every 6h (offset +1h from discovery) | 3.8 | 10m | Cron job running |

**Phase 3 Gate:** Enriched contact list with telephone-verified emails at 60%+ resolution rate

---

## Phase 4 — Outbound Channels

### SMTP Email Outreach
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.1 | Write email template engine (Jinja2 templates for cold outreach) | 1.10 | 30m | Template directory |
| 4.2 | Write personalized email generator: template + researcher context → custom body | 4.1 | 30m | `email_builder.py` |
| 4.3 | Write SMTP sender via Stalwart (Python smtplib, 587 TLS) | 4.2 | 30m | `smtp_sender.py` |
| 4.4 | Implement send rate limiter (respect warmup schedule, daily cap) | 4.3, 1.12 | 20m | Rate limiter |
| 4.5 | Add bounce detection + auto-unsubscribe mechanism | 4.4 | 30m | Bounce handler |
| 4.6 | Write Reply-To tracking setup (dedicated reply alias per campaign) | 4.5 | 20m | Reply tracking |

### Stack Exchange Integration
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.7 | Register Stack Exchange app (OAuth 2.0) | — | 15m | App key + secret |
| 4.8 | Generate and store OAuth access token | 4.7 | 10m | Access token |
| 4.9 | Write Stack Exchange questions.add wrapper | 4.8 | 45m | `stackexchange_post.py` |
| 4.10 | Implement site selection logic (match question topic to site) | 4.9 | 20m | Site routing |
| 4.11 | Write post template for Stack Exchange (academic-style query) | 4.10 | 20m | `se_templates/` |
| 4.12 | Add rate limiting (1 post per site per day max) | 4.9 | 15m | Rate limiter |

### Reddit Integration
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.13 | Create Reddit account + developer app (OAuth 2.0) | — | 15m | Client ID + secret |
| 4.14 | Write PRAW-based Reddit text post submitter | 4.13 | 30m | `reddit_post.py` |
| 4.15 | Implement subreddit routing (map topics to field-specific subs) | 4.14 | 20m | Subreddit mapping |
| 4.16 | Write post templates for each subreddit type | 4.15 | 20m | `reddit_templates/` |

### ResearchGate Integration
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.17 | Create ResearchGate account | — | 10m | Account |
| 4.18 | Write Playwright automation for RG Q&A posting | 4.17 | 60m | `researchgate_post.py` |
| 4.19 | Add cookie session persistence (avoid re-login every run) | 4.18 | 20m | Session file |

### Outbound Cron
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.20 | Write unified outbound script: read enriched candidates → send via best channel | 4.6, 4.12, 4.16, 4.19 | 45m | `expert_outbound.py` |
| 4.21 | Implement channel selection heuristics (email vs forum depending on nature of query) | 4.20 | 20m | Channel router |
| 4.22 | Register cron job: `outbound-outreach` every 12h | 4.20 | 10m | Cron job running |
| 4.23 | Register cron job: `response-monitor` every 6h | 4.6 | 10m | Cron job running |

**Phase 4 Gate:** Multi-channel outreach running — email, SE, Reddit, RG — with response tracking

---

## Phase 5 — Digitization Pipeline

### Archive Discovery
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 5.1 | Write Archive.org API query (check if item already digitized) | — | 30m | `archive_dot_org_check.py` |
| 5.2 | Write HathiTrust API query (partner institution content check) | — | 30m | `hathitrust_check.py` |
| 5.3 | Build finding-aid URL detection for university archives | — | 45m | `finding_aid_detector.py` |
| 5.4 | Write candidate generator: offline work → potential holding institutions | — | 30m | `holding_institution_matcher.py` |

### Digitization Requests
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 5.5 | Build university special collections web form detector + auto-filler | 5.3 | 60m | `digitization_webform.py` |
| 5.6 | Write email template for digitization request (item description, reason, rights context) | — | 20m | `digitization_email_template.md` |
| 5.7 | Write SMTP sender for digitization requests via digitization@yourdomain.com | 5.6, 4.3 | 20m | `digitization_sender.py` |
| 5.8 | Implement response tracking for digitization requests | 5.7 | 20m | Digitization tracker |

### Digitization Cron
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 5.9 | Write unified digitization script: offline work → check → request | 5.4, 5.7 | 30m | `digitization_pipeline.py` |
| 5.10 | Register cron job: `digitization-sweep` daily | 5.9 | 10m | Cron job running |

**Phase 5 Gate:** Automated digitization request submitted and tracked for an offline-identified work

---

## Phase 6 — Orchestration & Polish

### Response Management
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 6.1 | Write IMAP inbox poller for research@yourdomain.com | 1.10 | 30m | `inbox_poller.py` |
| 6.2 | Implement reply classification: positive/negative/need-info/OOO/spam | 6.1 | 45m | Reply classifier |
| 6.3 | Write follow-up sequencer (automatic polite nudge after N days of silence) | 6.2 | 30m | `followup_scheduler.py` |
| 6.4 | Add reputation monitoring dashboard (bounce rate, spam complaints, reply rate) | 6.2 | 30m | Reputation tracker |

### Integration & Skills
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 6.5 | Integrate discovery pipeline with intelligence-research-analyst skill | 2.13 | 30m | Calling convention |
| 6.6 | Write Hermes MCP-style tool for ad-hoc expert queries | All phases | 45m | `research_tools.py` |
| 6.7 | Create knowledge base of successful outreach templates | 4.2, 4.11, 4.16 | 30m | Template library |
| 6.8 | Add fact_store integration (store successful expert contacts in Holographic Memory) | All phases | 20m | Memory integration |

### Testing
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 6.9 | End-to-end: topic → discovered expert → email sent → reply tracked | All | 60m | Integration test |
| 6.10 | Dry-run mode for all outbound channels (log instead of send) | All | 30m | `--dry-run` flag |
| 6.11 | Bounce/scrape/error handling test suite | All | 45m | Test coverage |

**Phase 6 Gate:** Full pipeline running end-to-end with response tracking and dry-run mode

---

## Total Task Summary

| Phase | Tasks | Est. Time | Dependencies |
|-------|-------|-----------|-------------|
| 1 — Foundation | 12 tasks | ~2.5h | None |
| 2 — Discovery | 15 tasks | ~4.5h | Phase 1 |
| 3 — Enrichment | 9 tasks | ~3.5h | Phase 2 |
| 4 — Outbound | 17 tasks | ~6.5h | Phase 1, 3 |
| 5 — Digitization | 10 tasks | ~4.5h | Phase 1 |
| 6 — Orchestration | 11 tasks | ~5h | All |
| **Total** | **74 tasks** | **~26.5h** | |

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| Email deliverability (IP blacklisted, spam) | High | Medium | IP warmup, mail-tester verification, monitoring dashboard |
| Stack Exchange API rate-limited or revoked | Medium | Low | Multiple SE site rotation, Reddit fallback |
| ResearchGate blocks Playwright automation | Medium | Medium | Session cookie persistence, human-like timing |
| OpenAlex API changes/deprecation | High | Low | Semantic Scholar fallback, plan to maintain both |
| Hunter.io free tier too limited | Medium | High | Pattern-guessing + SMTP verify as primary, Hunter as enhancement |
| Cold email negative responses/reports | Medium | Medium | Polite templates, low volume, easy unsubscribe, reputation monitoring |
| No API for academic institutions' special collections | Medium | High | Universal email template as fallback for all web-form-less archives |

---

*Created 2026-07-21. See project brief for vision and architecture.*
