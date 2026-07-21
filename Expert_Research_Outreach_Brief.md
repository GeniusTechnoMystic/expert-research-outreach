# Expert Research & Outreach Infrastructure — Project Brief

**Status:** Design
**Created:** 2026-07-21
**Session:** Hermes research & planning session
**Related Skill:** `research/expert-research-outreach-infrastructure`
**Kanban slug:** `expert-research-outreach`

---

## 1. Vision

An infrastructure layer that lets Hermes reach beyond the digitized web — into human expertise, offline archives, and institutional collections. The agent identifies relevant experts, scholars, or academics for a given research question, determines how to contact them (email, forum, social media, expert network), and orchestrates the outreach. For offline data/works archives, it identifies digitization services and submits structured requests.

The core insight: not all knowledge is searchable online. The highest-value knowledge is often embedded in human experts' heads or sitting undigitized in library/museum archives. This project bridges that gap.

## 2. Core Capabilities

### 2.1 Researcher Discovery
- **Researcher identification:** Given a topic/question, find the top experts using OpenAlex API (300M+ papers, author metrics, institutional affiliations) and Semantic Scholar API (200M+ papers, citation networks)
- **Contact discovery:** Extract email addresses from OpenAlex metadata (~30% coverage), ORCID public profiles (~40%), and domain-based pattern guessing (first.last@institution.edu) verified via Hunter.io
- **Expert ranking:** Rank by h-index, recency of publications, topic relevance, citation velocity
- **Institutional affiliation:** Reverse-map via ROR ID to institution domain for contact discovery

### 2.2 Expert Outreach Channels
- **Email (self-hosted SMTP):** Cold-email researchers with templated, personalized queries via Stalwart mail server
- **Stack Exchange API:** Post research questions to relevant field-specific sites (Academia.SE, MathOverflow, Physics, Stats, etc.) with OAuth-authenticated API calls
- **Reddit API:** Post to field-specific subreddits (r/AskAcademia, r/AskHistorians, r/AskScience, discipline subs) via PRAW
- **ResearchGate Q&A:** Post "Ask a technical question" via Playwright browser automation (no API available)
- **Paid expert networks:** Via CleverX API (cheaper tier, ~$50-200/call) or Guidepoint MCP integration for premium expert access

### 2.3 Archive & Digitization Pipeline
- **Digitization status check:** Query Archive.org API, HathiTrust API, and institutional repositories to check if a given work is already digitized
- **Digitization request:** Submit structured digitization requests to university special collections departments (web forms or templated email)
- **Commercial digitization:** Route to services like BSLW for paid digitization when needed
- **Finding aid navigation:** Parse archival finding aids to identify relevant collections and their digitization status

### 2.4 Response Management
- **Reply-to monitoring:** Track inbound responses to outreach emails (via IMAP/JMAP to Stalwart)
- **Response classification:** Categorize as positive, negative, need-more-info, or out-of-office
- **Follow-up sequencing:** Automated polite follow-ups after configurable intervals
- **Reputation management:** IP warmup scheduling, bounce handling, spam complaint monitoring

## 3. Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                    RESEARCH ORCHESTRATOR                         │
│                    (Hermes cron pipeline)                        │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐   ┌─────────────────┐   ┌───────────────┐  │
│  │    DISCOVERY     │──▶│   ENRICHMENT    │──▶│   OUTBOUND    │  │
│  │  (every 6h)      │   │  (every 6h+1h)  │   │  (every 12h)  │  │
│  │                  │   │                  │   │               │  │
│  │ OpenAlex API     │   │ Hunter.io verify │   │ Stalwart SMTP │  │
│  │ Semantic Scholar │   │ ORCID lookup     │   │ StackExchange │  │
│  │ Topic filtering  │   │ Pattern guessing │   │ Reddit PRAW   │  │
│  │ Authority rank   │   │ Affil→domain     │   │ ResearchGate  │  │
│  └─────────────────┘   └─────────────────┘   └───────────────┘  │
│                        │                                          │
│                        ▼                                          │
│               ┌─────────────────┐                                │
│               │   DIGITIZATION   │  (daily)                       │
│               │                  │                                │
│               │ Archive.org API  │                                │
│               │ HathiTrust API   │                                │
│               │ Univ. web forms  │                                │
│               │ Email requests   │                                │
│               └─────────────────┘                                │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐
│   EMAIL INFRA    │  │  FORUM ACCOUNTS │  │  EXPERT NETWORKS │
│  Stalwart (Rust) │  │  StackExchange  │  │  CleverX API     │
│  Postfix-optional│  │  Reddit         │  │  Guidepoint MCP  │
│  DKIM/DMARC/SPF  │  │  ResearchGate   │  │  (future)        │
│  JMAP + IMAP     │  │  LinkedIn       │  │                  │
│  Catch-all       │  │  (browser)      │  │                  │
└──────────────────┘  └─────────────────┘  └──────────────────┘
```

## 4. Key Components

### 4.1 Email Infrastructure
- **Stalwart Mail Server** — Rust-based, single binary, ~180MB RAM, native NixOS module
- **VPS** — Hetzner CX22 ($5.50/mo, port 25 open) or co-locate on existing infra
- **Domain** — custom domain with MX, SPF, DKIM, DMARC, MTA-STS
- **Warmup schedule** — progressive volume ramp from 5→100 emails/day over 8 weeks

### 4.2 Discovery Layer
- **OpenAlex API** — `GET /authors?search=<topic>`, filter by works_count, cited_by_count, 2yr_mean_citedness
- **Semantic Scholar API** — `GET /paper/search?query=<topic>`, author details, citation graph
- **ORCID API** — `GET /{orcid}/record` for public email and affiliation data
- **Hunter.io API** — `GET /email-finder` for pattern-based email verification

### 4.3 Forum Integration
- **Stack Exchange API** — OAuth 2.0, registered app, `POST /questions/add`
- **Reddit API (PRAW)** — OAuth 2.0, `subreddit.submit()` for text posts
- **ResearchGate** — Playwright Python browser automation for Q&A posting
- **LinkedIn** — Playwright automation for professional networking

### 4.4 Cron Pipeline
- **Cron 1 (6h):** `expert-discovery` — runs OpenAlex/SemanticScholar queries → writes candidate list
- **Cron 2 (6h+1h):** `expert-enrich` — runs Hunter.io + ORCID + pattern guess on candidates
- **Cron 3 (12h):** `outbound-outreach` — sends templated emails + forum posts (low volume)
- **Cron 4 (daily):** `digitization-sweep` — checks archive.org, submits digitization requests
- **Cron 5 (daily):** `response-monitor` — polls inbox for replies, classifies, triggers follow-ups

## 5. Phases

| Phase | Focus | Duration | Deliverable |
|-------|-------|----------|-------------|
| **1 — Foundation** | Domain + VPS + Stalwart email + DNS | Week 1 | Working self-hosted email, verified deliverability |
| **2 — Discovery** | OpenAlex + Semantic Scholar + ORCID integration | Week 2 | Discovery cron that finds experts for any topic |
| **3 — Enrichment** | Hunter.io + email pattern guessing + verification | Week 2 | Enriched contact list with verified emails |
| **4 — Outbound** | Stack Exchange + Reddit + SMTP outreach | Week 3 | Multi-channel automated expert querying |
| **5 — Digitization** | Archive.org + university form detection | Week 3 | Automated digitization request pipeline |
| **6 — Orchestration** | Response handling + reputation + integration | Week 4 | Full pipeline with response classification |

## 6. Cost Estimates

| Item | Monthly | Annual |
|------|---------|--------|
| Domain registration | — | $10-15/yr |
| Hetzner VPS CX22 | $5.50/mo | $66/yr |
| Hunter.io (starter, optional) | $49/mo | $588/yr |
| CleverX API (optional) | ~$100/mo | ~$1,200/yr |
| **Minimum viable** | **$5.50/mo** | **$76-81/yr** |
| **Full stack** | **~$155/mo** | **~$1,860/yr** |

## 7. Related

- **Research skill:** `intelligence-research-analyst` — discovery methodology
- **Infrastructure skill:** `expert-research-outreach-infrastructure` — this project
- **System skills:** `nixos-development`, `cron-script-hygiene`, `systems-engineering-best-practices`

---

*Created 2026-07-21. Based on comprehensive research of expert networks, academic social platforms, academic social networks, digitization services, and self-hosted email infrastructure.*
