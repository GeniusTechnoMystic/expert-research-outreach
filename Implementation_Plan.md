# Expert Research & Outreach Implementation Plan (v2.0)

**Status:** Design / Not Started
**Created:** 2026-07-21 | **Updated:** 2026-07-21 (post-review)
**Project path:** `~/workspace/documents/Projects/Expert-Research-Outreach/`
**Related skill:** `research/expert-research-outreach-infrastructure`
**Kanban slug:** `expert-research-outreach`
**Review source:** AION Meta-Strategist review — 13 findings incorporated

---

## CONSOLIDATED CHANGE LOG

| # | Finding | Severity | Action Taken |
|---|---------|----------|-------------|
| 1 | Hard circuit breaker for email reputation | CRITICAL | Added to Phase 1 — auto-stop on bounce >3% or any spam complaint |
| 2 | TOS violations from automated forum posting | CRITICAL | Added disclosure language req, human review gate, rate limits to Phase 4A |
| 3 | Queue discipline between phases | CRITICAL | Added shared priority queue (Phase 6.0), moved earlier in sequence |
| 4 | Missing feedback loop (replies → ranking) | HIGH | Added feedback loop to Phase 6, architecture diagram updated |
| 5 | Phase 4 (LinkedIn/researchgate) unrealistic as single block | HIGH | Split Phase 4 into 4A (API-based) + 4B (browser automation) |
| 6 | Phase 5 placed too late | HIGH | Moved Phase 5 to run parallel with Phase 2 |
| 7 | No human review gate for early outbound | HIGH | Added Phase 3.5 — Review Gate with auto-promotion |
| 8 | LinkedIn API restrictions | HIGH | Noted LinkedIn InMail is partner-only; replaced with DM/nurture pattern |
| 9 | No recipient-side value proposition | MEDIUM | Added to Brief §2.0 |
| 10 | Template quality tracking missing | MEDIUM | Added A/B test task + response-rate analysis cron |
| 11 | ORCID rate limits not documented | MEDIUM | Added rate-limit tracking to ORCID integration tasks |
| 12 | Calendar timeline should note 8-week warmup | MEDIUM | Updated timeline to reflect 6-8 calendar weeks |
| 13 | Domain strategy / subdomain evaluation | MEDIUM | Added domain selection criteria to Phase 1 |

---

## Phase 1 — Foundation: Email Infrastructure

### Domain & DNS Setup
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 1.1 | **Domain strategy:** Evaluate existing domain vs new registration. Prefer `.org`/`.io` over `.com`. Consider subdomain of existing domain (better sending reputation inheritance). | — | 20m | Domain decision doc |
| 1.2 | Register domain (e.g., `yourresearch.org`) | 1.1 | 15m | Registrar account + domain |
| 1.3 | Point domain nameservers to VPS DNS | 1.2 | 10m | NS records set |
| 1.4 | Set MX, A, SPF, DKIM, DMARC, MTA-STS records | 1.3 | 20m | DNS zone file |
| 1.5 | Set PTR/rdns record at VPS provider | 1.4 | 10m | Reverse DNS match |

### VPS Provisioning
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 1.6 | Provision Hetzner CX22 (or other, port 25 unblocked) | — | 15m | VPS running NixOS |
| 1.7 | Write IP warmup script with **HARD circuit breaker**: auto-stop if bounce >3% or ANY spam complaint received. Logs delivery metrics per batch. | 1.6 | 45m | `warmup.py` with circuit breaker |
| 1.8 | Configure firewall (port 25, 465, 587, 993, 8080) | 1.6 | 10m | iptables/nftables rules |
| 1.9 | Set up ACME TLS (Let's Encrypt for mail subdomain) | 1.4 | 10m | SSL certs for mail.yourdomain.com |

### Stalwart Deployment
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 1.10 | Write Stalwart NixOS module config | 1.6 | 30m | `services.stalwart` stanza |
| 1.11 | Generate DKIM keys, configure DNS signing | 1.4, 1.10 | 15m | DKIM TXT record |
| 1.12 | Create mail accounts (research, postmaster, abuse, digitization, forum aliases) | 1.10 | 10m | Working mailboxes |
| 1.13 | Verify deliverability (mail-tester.com, send test to Gmail/Outlook) | 1.12 | 20m | 10/10 score |

**Phase 1 Gate:** Sending and receiving email from research@yourdomain.com with 10/10 mail-tester score. Warmup circuit breaker tested and armed.

---

## Phase 2 — Discovery Pipeline (runs parallel with Phase 5)

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
| 2.10 | Write ORCID API query (public tier: 4,000 req/day, 8,000 with key) | — | 30m | `orcid_lookup.py` with rate-limit tracking |
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

## Phase 3.5 — Human Review Gate (NEW)

Before any outbound automation runs, all messages must pass through a human review queue. The gate auto-disengages after N successful batches with 0 complaints.

| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 3.10 | Design review queue data model: templates, personalized messages, destination channel, scheduled send time | 3.8 | 20m | Review queue schema |
| 3.11 | Write review queue writer: outbound messages go to `~/.hermes/data/review_queue/` instead of being sent | 3.10 | 30m | `review_queue.py` |
| 3.12 | Build review approval interface: user reviews batch, approves/edit/rejects per message | 3.11 | 45m | Approval CLI |
| 3.13 | Implement auto-promotion: after 5 batches with 0 complaints, gate disengages and messages send directly | 3.12 | 20m | Auto-promotion logic |
| 3.14 | Add TOS compliance disclosure to all templates: attribution line ("This query was composed with AI assistance for research purposes"), clear opt-out, non-commercial framing | 3.11 | 20m | Compliance footer |

**Phase 3.5 Gate:** First outbound batch reviewed and approved by user

---

## Phase 4A — Outbound: API-Based Channels (NEW — split from original Phase 4)

### SMTP Email Outreach
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.1 | Write email template engine (Jinja2 templates for cold outreach) | 1.12 | 30m | Template directory |
| 4.2 | Write personalized email generator: template + researcher context → custom body, demonstrates prior-work awareness | 4.1 | 30m | `email_builder.py` |
| 4.3 | Write SMTP sender via Stalwart (Python smtplib, 587 TLS) | 4.2 | 30m | `smtp_sender.py` |
| 4.4 | Implement send rate limiter (respect warmup schedule, daily cap, circuit breaker) | 4.3, 1.7 | 20m | Rate limiter |
| 4.5 | Add bounce detection + auto-unsubscribe mechanism | 4.4 | 30m | Bounce handler |
| 4.6 | Write Reply-To tracking setup (dedicated reply alias per campaign) | 4.5 | 20m | Reply tracking |

### Stack Exchange Integration
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.7 | Register Stack Exchange app (OAuth 2.0) | — | 15m | App key + secret |
| 4.8 | Generate and store OAuth access token | 4.7 | 10m | Access token |
| 4.9 | Write Stack Exchange questions.add wrapper with disclosure in post body | 4.8 | 45m | `stackexchange_post.py` |
| 4.10 | Implement site selection logic (match question topic to site) | 4.9 | 20m | Site routing |
| 4.11 | Write post template for Stack Exchange (academic-style query with attribution) | 4.10 | 20m | `se_templates/` |
| 4.12 | Add rate limiting (1 post per site per day max, human-like posting schedule) | 4.9 | 15m | Rate limiter |

### Reddit Integration
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.13 | Create Reddit account + developer app (OAuth 2.0) | — | 15m | Client ID + secret |
| 4.14 | Write PRAW-based Reddit text post submitter with attribution in post | 4.13 | 30m | `reddit_post.py` |
| 4.15 | Implement subreddit routing (map topics to field-specific subs) | 4.14 | 20m | Subreddit mapping |
| 4.16 | Write post templates for each subreddit type | 4.15 | 20m | `reddit_templates/` |

### Outbound Cron (4A)
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.17 | Write unified outbound-4A script: read enriched candidates → send via email/SE/Reddit | 4.6, 4.12, 4.16 | 45m | `expert_outbound_4a.py` |
| 4.18 | Implement channel selection heuristics (email vs forum depending on nature of query) | 4.17 | 20m | Channel router |
| 4.19 | Register cron job: `outbound-outreach-4a` every 12h | 4.17 | 10m | Cron job running |

**Phase 4A Gate:** Email, SE, and Reddit outreach running with disclosure, rate limiting, and review gate

---

## Phase 4B — Outbound: Browser Automation (NEW — LinkedIn+ResearchGate)

### ResearchGate Integration
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.20 | Create ResearchGate account | — | 10m | Account |
| 4.21 | Write Playwright automation for RG Q&A posting | 4.20 | 60m | `researchgate_post.py` |
| 4.22 | Add cookie session persistence (avoid re-login every run) | 4.21 | 20m | Session file |
| 4.23 | Add CAPTCHA detection + notification (pause and alert user, don't attempt bypass) | 4.21 | 20m | CAPTCHA handler |

### LinkedIn Integration (SCOPE REDUCED)
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.24 | Research LinkedIn API restrictions: InMail is **partner-only** (requires approved Developer Program). Document alternatives: connection request + DM, LinkedIn Sales Navigator, browser automation with limitations. | — | 30m | LinkedIn strategy doc |
| 4.25 | **IF feasible:** Write Playwright automation for LinkedIn connection requests + follow-up DMs | 4.24 | 60m | `linkedin_connect.py` |
| 4.26 | **IF NOT feasible:** Implement alternative: academic-professional network DM via ResearchGate (already covers this use case) | 4.24 | 15m | Fallback note |

### Outbound Cron (4B)
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 4.27 | Write unified outbound-4B script | 4.23, 4.25/4.26 | 30m | `expert_outbound_4b.py` |
| 4.28 | Register cron: `outbound-outreach-4b` daily (lower frequency — browser automation is fragile) | 4.27 | 10m | Cron job running |

**Phase 4B Gate:** Browser-automated channels running with CAPTCHA handling and fallback strategy

---

## Phase 5 — Digitization Pipeline (runs parallel with Phase 2)

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

## Phase 6 — Orchestration, Feedback & Quality

### Phase 6.0 — Shared Queue Discipline (NEW — prerequisite for all outbound)
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 6.0.1 | Design shared priority queue: consumed by all outbound crons, with aging, dedup, and throttle | 3.8 | 30m | Queue schema |
| 6.0.2 | Write queue manager: discovery → enrichment → queue → throttle → outbound | 6.0.1 | 45m | `outbound_queue.py` |
| 6.0.3 | Implement backpressure: if any downstream phase is backed up, upstream cron pauses producing | 6.0.2 | 20m | Backpressure logic |

### Response Management
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 6.1 | Write IMAP inbox poller for research@yourdomain.com | 1.12 | 30m | `inbox_poller.py` |
| 6.2 | Implement reply classification: positive/negative/need-info/OOO/spam | 6.1 | 45m | Reply classifier |
| 6.3 | Write follow-up sequencer (automatic polite nudge after N days of silence) | 6.2 | 30m | `followup_scheduler.py` |
| 6.4 | Add reputation monitoring dashboard (bounce rate, spam complaints, reply rate) | 6.2 | 30m | Reputation tracker |

### Feedback Loop — Replies → Ranking (NEW)
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 6.5 | Write reply feedback consumer: positive reply → boost researcher's future ranking score; negative/no-reply → decay | 6.2 | 30m | `reply_feedback.py` |
| 6.6 | Integrate feedback into discovery ranking: update OpenAlex/Semantic Scholar scores with engagement signal | 6.5, 2.13 | 20m | Feedback-weighted ranking |
| 6.7 | Register cron job: `feedback-integrate` daily | 6.6 | 10m | Cron job running |

### Template Quality Tracking (NEW)
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 6.8 | Build response-rate tracker: per-template, per-channel reply rate | 6.2 | 30m | Template analytics |
| 6.9 | Implement A/B test framework: rotate template variants, compare response rates | 6.8 | 30m | Template A/B engine |
| 6.10 | Write auto-selector: after N samples, auto-select highest-performing template variant | 6.9 | 20m | Template optimizer |

### Integration & Skills
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 6.11 | Integrate discovery pipeline with intelligence-research-analyst skill | 2.13 | 30m | Calling convention |
| 6.12 | Write Hermes MCP-style tool for ad-hoc expert queries | All phases | 45m | `research_tools.py` |
| 6.13 | Create knowledge base of successful outreach templates | 4.2, 4.11, 4.16 | 30m | Template library |
| 6.14 | Add fact_store integration (store successful expert contacts in Holographic Memory) | All phases | 20m | Memory integration |

### Testing
| # | Task | Dependencies | Est. Time | Output |
|---|------|-------------|-----------|--------|
| 6.15 | End-to-end: topic → discovered expert → email sent → reply tracked | All | 60m | Integration test |
| 6.16 | Dry-run mode for all outbound channels (log instead of send) | All | 30m | `--dry-run` flag |
| 6.17 | Bounce/scrape/error handling test suite | All | 45m | Test coverage |

**Phase 6 Gate:** Full pipeline running end-to-end with feedback loop, template optimization, and dry-run mode

---

## Total Task Summary

| Phase | Tasks | Est. Dev Time | Calendar Time | Dependencies |
|-------|-------|-------------|---------------|-------------|
| 1 — Foundation | 13 tasks | ~3h | Week 1-2 | None |
| 2 — Discovery | 15 tasks | ~4.5h | Week 2-3 | Phase 1 |
| 3 — Enrichment | 9 tasks | ~3.5h | Week 3 | Phase 2 |
| 3.5 — Review Gate | 5 tasks | ~2h | Week 3 | Phase 3 |
| 4A — Outbound API | 12 tasks | ~4.5h | Week 3-4 | Phase 1, 3, 3.5 |
| 4B — Browser Auto | 9 tasks | ~3.5h | Week 4-5 | Phase 4A |
| 5 — Digitization | 10 tasks | ~4.5h | Week 2-4 (parallel 2) | Phase 1 |
| 6 — Orchestration | 17 tasks | ~7.5h | Week 5-8 | All |
| **Total** | **90 tasks** | **~33h** | **6-8 calendar weeks** | |

**Note on calendar time:** IP warmup requires 8 weeks regardless of dev effort. Weeks 5-8 can overlap with Phase 6 work. The 6-8 week estimate accounts for DNS propagation delays, warmup constraints, and Playwright debug time.

---

## Risk Register (v2.0 — expanded)

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| Email deliverability (IP blacklisted, spam) | **Critical** | Medium | **Circuit breaker:** auto-stop on bounce >3% or any spam complaint. IP warmup schedule (8 weeks). mail-tester verification. Monitoring dashboard. Separate IP for email (don't colocate with other services). |
| TOS violation — automated forum posting | **High** | Medium-High | Disclosure in every post body. Human review gate (Phase 3.5). Conservative rate limits (1 post/site/day). Human-like posting schedule (not at clock intervals). Track TOS of each platform independently. |
| Queue overflow — phases out of sync | **High** | Medium | Shared priority queue (Phase 6.0) with backpressure. Cron jobs check queue capacity before producing. Aging ensures old candidates don't get stuck. |
| ResearchGate blocks Playwright automation | Medium | Medium | Session cookie persistence, human-like timing, CAPTCHA pause + notify. Fallback: no automation, manual posting via email template. |
| OpenAlex API changes/deprecation | High | Low | Semantic Scholar fallback. Plan to maintain both APIs. Local cache of recent results. |
| Hunter.io free tier too limited | Medium | High | Pattern-guessing + SMTP verify as primary. Hunter as enhancement only. Upgrade to paid only if free tier is bottleneck. |
| Cold email negative responses/reports | Medium | Medium | Polite templates, low volume, easy unsubscribe, attribution disclosure, reputation monitoring, circuit breaker. |
| No API for academic institutions' special collections | Medium | High | Universal email template as fallback. Web form auto-filler for institutions that have them. |
| GDPR/Privacy compliance — processing scraped emails | Medium | Low | Privacy notice in email footer. One-click unsubscribe. Documented legitimate-interest assessment. Data retention limits (delete after 90 days). |
| Domain reputation death before warmup matures | **Critical** | Medium | Separate VPS (clean IP). Warmup from 5/day. Circuit breaker on any negative signal. Consider using subdomain of existing established domain. |
| LinkedIn API restrictions (partner-only) | Medium | High | Investigated in Phase 4B. If blocked, fall back to ResearchGate + email for the same use case. |

---

## Domain Strategy

| Factor | Recommendation |
|--------|---------------|
| TLD | `.org` or `.io` preferred over `.com` for academic outreach |
| Sending reputation | New domain = zero reputation. Consider subdomain of existing domain (e.g., `research.existingdomain.com`) if available |
| Warmup | 8 weeks minimum, 5→100 emails/day, circuit breaker armed |
| Subdomain evaluation | `research.yourdomain.com` vs `yourresearch.org` — subdomain inherits some domain reputation, faster warmup |
| Registrar | Namecheap, Porkbun, or Cloudflare (no markup, easy DNS management) |

---

*Updated 2026-07-21. v2.0 changes: 90 tasks (was 74), 6-8 week calendar timeline (was 4), 4 new phases (3.5, 4A, 4B, 6.0), queue discipline, circuit breaker, feedback loop, review gate, expanded risk register.*