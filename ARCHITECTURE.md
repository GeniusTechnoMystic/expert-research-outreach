# Expert Research & Outreach — Architecture Design Document

**Status:** Design / Not Started
**Version:** 2.0
**Created:** 2026-07-21
**System:** Automated expert/academic outreach — email, forum Q&A, digitization requests
**Metapatterns:** Pipeline (primary) + Orchestrator (coordination) + Plugin (channel adapters)
**Key files:**
  - `Expert_Research_Outreach_Brief.md` — vision, scope, channel analysis
  - `Implementation_Plan.md` — 90 tasks, 8 phases, risk register
  - `README.md` — project landmark

---

## 1. System Context

### 1.1 Problem Domain

Not all knowledge is searchable online. The highest-value information is often:
- **Embedded in human expertise** — a researcher who has spent 20 years studying a specific phenomenon
- **Sitting in offline archives** — library/museum special collections, institutional repositories, unpublished datasets
- **Distributed across academic networks** — preprint servers, conference proceedings, lab notebooks

Current tools (web search, academic search engines, LLMs trained on public text) can only reach the digitized, publicly-indexed fraction. The rest requires **direct human contact** — identifying who knows, and reaching them.

### 1.2 System Scope

| In Scope | Out of Scope |
|----------|-------------|
| Identifying top researchers for a given topic via OpenAlex + Semantic Scholar | Building a general-purpose search engine |
| Discovering contact information (email, affiliation) via ORCID, Hunter.io, pattern guessing | Scraping non-public contact databases |
| Sending cold-email outreach from a self-hosted email server | Spam / unsolicited bulk marketing |
| Posting research questions to Stack Exchange, Reddit, ResearchGate | Automating social media content creation |
| Requesting digitization of offline works from archive holders | Physically digitizing materials |
| Tracking responses, follow-ups, and reputation metrics | Maintaining a CRM system |

### 1.3 Stakeholders

| Stakeholder | Interest |
|-------------|----------|
| **User (researcher)** | Has a research question; needs answers from experts or access to offline materials |
| **Target expert** | Receives the query; needs to perceive it as legitimate, low-friction, and worth responding to |
| **Self-hosted domain** | Email reputation is at stake; must not be burned by spam complaints |
| **Platform operators** (Stack Exchange, Reddit, etc.) | TOS governs automated posting; must not be violated |

---

## 2. Architecture Topology

### 2.1 Metapattern Decomposition

The system uses three metapatterns:

| Metapattern | Role | Where |
|-------------|------|-------|
| **Pipeline** | Primary — data flows through sequential stages (discover → enrich → review → outbound) | The cron chain: each stage writes structured output consumed by the next |
| **Orchestrator** | Coordination — a central scheduler manages inter-stage timing, queue discipline, and backpressure | The shared priority queue (Phase 6.0) and cron orchestrator |
| **Plugin** | Channel adapters — each outbound channel (SMTP, Stack Exchange, Reddit, ResearchGate) is a pluggable module behind a uniform interface | Outbound 4A/4B channel implementations |

### 2.2 Layered View

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ORCHESTRATOR LAYER                             │
│  Cron scheduler  │  Priority queue  │  Backpressure  │  Circuit     │
│                   │  (aging, dedup)  │  controller    │  breaker     │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│                        PIPELINE LAYER                                 │
│                                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │Discovery │─▶│Enrichment│─▶│  Review  │─▶│Outbound  │─▶│Feedback│  │
│  │Phase 2   │  │Phase 3   │  │Gate 3.5  │  │Phases 4A │  │Phase 6 │  │
│  │          │  │          │  │          │  │  + 4B    │  │        │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └────────┘  │
│                                                                       │
│  ┌──────────┐  (Phase 5 runs parallel with Phase 2)                   │
│  │Digitztn  │                                                        │
│  │Phase 5   │                                                        │
│  └──────────┘                                                        │
└─────────────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│                        INFRASTRUCTURE LAYER                           │
│                                                                       │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐ │
│  │   Stalwart Mail   │  │   OpenAlex API     │  │  Semantic Scholar  │ │
│  │   (SMTP/IMAP)     │  │   (REST)           │  │  API (REST)        │ │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘ │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐ │
│  │   Stack Exchange  │  │   Reddit API      │  │   Hunter.io       │ │
│  │   API (OAuth)     │  │   (PRAW/OAuth)    │  │   API (REST)      │ │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘ │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐ │
│  │   ORCID API       │  │   Archive.org API │  │   Playwright      │ │
│  │   (REST)          │  │   (REST)          │  │   (Browser)       │ │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Data Flow — End-to-End

```
User query: "What is the current state of research on X?"

1. DISCOVERY (every 6h)
   OpenAlex API: /authors?search=X → ranked author list (h-index, recency, citation velocity)
   Semantic Scholar: /paper/search?query=X → author details, citation graph
   ORCID: /{orcid}/record → public email + affiliation
   ↓
   Output: /var/data/research_candidates/{run_id}/candidates.json
   Schema: [{id, name, email?, orcid, affiliation, rank_fields, source}]

2. ENRICHMENT (6h+1h offset)
   Hunter.io: domain + name → verified email
   Pattern guesser: first.last@institution.edu → candidate emails
   SMTP verify: HELO check on MX server (no delivery)
   Priority queue: metadata > ORCID > Hunter.io > pattern > SMTP verify
   ↓
   Output: /var/data/research_candidates/{run_id}/enriched.json
   Schema: [{id, name, email, confidence_level, source, verification_method}]

3. REVIEW GATE (Phase 3.5)
   All outbound messages written to review queue
   User approves/edits/rejects batch
   After 5 clean batches with 0 complaints → auto-promote
   TOS compliance footer appended to all messages
   ↓
   Output: /var/data/outbound_queue/{run_id}/approved.json

4. OUTBOUND 4A (every 12h, API-based)
   SMTP: piped through Stalwart (587 TLS, rate-limited, signed DKIM)
   Stack Exchange: POST /questions/add (OAuth, attribution footer)
   Reddit: subreddit.submit() (PRAW, attribution)
   Rate limiter: 1 post/site/day, 10-20 emails/day (warmup schedule)
   Circuit breaker: STOP if bounce >3% or any spam complaint
   ↓
   Output: /var/data/outbound_log/{run_id}/sent.json

5. OUTBOUND 4B (daily, browser automation)
   ResearchGate: Playwright → Q&A post (cookie session, CAPTCHA detection)
   ↓
   Output: /var/data/outbound_log/{run_id}/browser_sent.json

6. FEEDBACK LOOP (daily)
   IMAP poller: check research@ for replies
   Classifier: positive / negative / need-info / OOO / spam
   Positive → boost researcher's future ranking score
   Negative / no-reply → decay
   Follow-up: polite nudge after N days (configurable)
   ↓
   Output: /var/data/feedback/{run_id}/classified.json
   → Updates discovery ranking weights

7. DIGITIZATION (daily, parallel with Phase 2)
   Archive.org API: check if item already digitized
   HathiTrust API: check partner institution content
   Finding aid detector: locate offline collections
   Email or web form: submit digitization request
   ↓
   Output: /var/data/digitization/{run_id}/requests.json
```

---

## 3. Component Design

### 3.1 Discovery Engine

```
Component: Discovery
Trigger: Cron (every 6h)
Input: Topic query string
Output: Ranked candidate list
```

**Interface:**

```python
class DiscoveryEngine:
    def discover(topic: str, filters: dict) -> list[Candidate]:
        """
        Query OpenAlex and Semantic Scholar for researchers matching topic.
        Merges, deduplicates, and ranks by authority metrics.
        """
        pass

    @dataclass
    class Candidate:
        id: str              # OpenAlex ID or Semantic Scholar ID
        name: str
        email: str | None    # from OpenAlex metadata if available
        orcid: str | None
        affiliation: str
        institution_domain: str | None  # from ROR ID
        h_index: int
        works_count: int
        cited_by_count: int
        two_year_mean: float
        topic_relevance: float  # 0.0 - 1.0
        source: str           # "openalex" | "semanticscholar" | "both"
```

**API endpoints used:**
- `GET https://api.openalex.org/authors?search={topic}&filter=works_count:>10&sort=cited_by_count:desc`
- `GET https://api.semanticscholar.org/graph/v1/paper/search?query={topic}&limit=100&fields=title,authors,year,citationCount`
- `GET https://pub.orcid.org/v3.0/{orcid}/record` (rate limit: 4,000 req/day anonymous, 8,000 with key)

**Error handling:**
- OpenAlex API failure → fall back to Semantic Scholar only, log warning
- Both APIs fail → skip cycle, alert via cron notification
- Rate limit hit → back off, resume next cycle (queue preserves state)

### 3.2 Enrichment Pipeline

```
Component: Enrichment
Trigger: Cron (6h+1h offset)
Input: Candidate list from Discovery
Output: Enriched candidate list with verified email
```

**Interface:**

```python
class EnrichmentPipeline:
    def enrich(candidates: list[Candidate]) -> list[EnrichedCandidate]:
        """
        Attempt to find verified email for each candidate.
        Priority: metadata > ORCID > Hunter.io > pattern guess > SMTP verify
        """
        pass

    @dataclass
    class EnrichedCandidate:
        id: str
        name: str
        email: str | None
        confidence: float    # 0.0 - 1.0 based on verification method
        verification_method: str  # "metadata" | "orcid" | "hunter" | "pattern" | "smtp_verify"
        alt_emails: list[str]
```

**Priority queue heuristic:**
| Method | Confidence | Cost | Coverage |
|--------|-----------|------|----------|
| OpenAlex metadata email | 0.95 | Free | ~30% |
| ORCID public email | 0.90 | Free | ~40% |
| Hunter.io verified | 0.85 | $49/mo (starter) | Varies |
| Pattern guess + SMTP verify | 0.70 | Free | ~60% |
| Pattern guess only | 0.40 | Free | ~80% |

> **⚠️ SMTP verify risk:** HELO checks against MX servers from an unknown IP can be interpreted as spam reconnaissance. Some providers (Microsoft, Google) track connection patterns from new IPs. **SMTP verify is disabled by default.** Enable manually only after domain reputation is established (week 8+). During warmup, rely on Hunter.io + pattern guess only. See DESIGN_REVIEW.md Finding 6.

### 3.3 Outbound Channel Adapters (Plugin Pattern)

Each channel implements a uniform interface:

```python
class OutboundChannel(ABC):
    @abstractmethod
    def send(self, message: Message) -> SendResult:
        """
        Send a message through this channel.
        Returns success/failure with delivery metadata.
        """
        pass

    @abstractmethod
    def can_send(self, message: Message) -> bool:
        """
        Check if this channel is appropriate for the given message.
        """
        pass

    @abstractmethod
    def remaining_quota(self) -> int:
        """
        Remaining sends for current period (rate limit aware).
        """
        pass

    @dataclass
    class Message:
        to: str
        subject: str
        body: str
        attachments: list[str] | None
        channel_specific: dict | None  # e.g. {"site": "academia.stackexchange.com"}

    @dataclass
    class SendResult:
        success: bool
        message_id: str | None
        error: str | None
        rate_limited: bool
        delivered_at: datetime
```

**Channel Registry:**

| Channel | Class | Method | Rate Limit | Auth |
|---------|-------|--------|-----------|------|
| SMTP (Stalwart) | `SMTPChannel` | `smtplib` TLS 587 | 10-100/day (warmup) | SMTP credentials |
| Stack Exchange | `SEChannel` | REST API | 1 post/site/day | OAuth 2.0 |
| Reddit | `RedditChannel` | PRAW | 1 post/sub/day | OAuth 2.0 |
| ResearchGate | `RGChannel` | Playwright | 1 post/day | Cookie session |

### 3.4 Rate Limiter

```python
class RateLimiter:
    """
    Centralized rate limiting across all outbound channels.
    Tracks per-channel send counts per time window.
    Enforces warmup curve for SMTP (Phase 1 schedule).
    """
    def remaining_quota(self, channel: str) -> int:
        """Sends remaining for current time window."""
        pass

    def consume(self, channel: str, count: int = 1) -> bool:
        """Attempt to consume N sends. Returns False if over quota."""
        pass

    def warmup_stage(self) -> int:
        """Returns current warmup week number (1-8+)."""
        pass

    def reset_at(self, channel: str) -> datetime:
        """When does the current quota window reset?"""
        pass
```

### 3.5 Queue Manager

```python
class PriorityQueue:
    """
    Shared queue connecting all pipeline stages.
    Implements backpressure, aging, dedup, and throttle.
    """
    def enqueue(self, item: QueueItem) -> None: ...
    def dequeue(self, max_items: int) -> list[QueueItem]: ...
    def age_items(self) -> None:  # boost priority of old items
    def dedup_by_email(self, email: str) -> bool: ...
    def backpressure_level(self) -> float:  # 0.0 = idle, 1.0 = full
    def get_queue_depth(self, channel: str) -> int: ...
    def get_oldest_item_age(self) -> timedelta: ...
    def retry_failed(self, item_id: str) -> None: ...
    def purge_expired(self, max_age: timedelta) -> int: ...
```

**Queue discipline:**
- Items age by 1 priority point per 24h (stale items don't get stuck)
- Dedup by email address (don't contact the same person twice)
- Discovery cron checks `backpressure_level()` before producing — if >0.8, skip cycle
- Outbound cron checks `remaining_quota()` before consuming — respects warmup

### 3.5 Circuit Breaker

```python
class CircuitBreaker:
    """
    Monitors email delivery metrics.
    HARD STOP on any spam complaint or bounce >3%.
    """
    state: "CLOSED" | "OPEN" | "HALF_OPEN"

    def record_bounce(self, email: str, reason: str) -> None: ...
    def record_complaint(self, email: str, source: str) -> None: ...
    def record_delivery(self, email: str, success: bool) -> None: ...

    def is_allowed(self) -> bool:
        """Check if outbound is allowed."""
        pass

    def auto_reset(self) -> None:
        """
        Automatically reset to HALF_OPEN after cooldown period.
        Send a single test email. If it succeeds, CLOSED. If not, OPEN.
        """
        pass
```

**Thresholds:**
| Metric | Threshold | Action |
|--------|-----------|--------|
| Spam complaint | ANY | OPEN immediately, alert user |
| Bounce rate | >3% | OPEN immediately, alert user |
| Bounce rate | 1-3% | WARN, log, continue |
| Bounce rate | <1% | Normal operation |

### 3.7 Feedback Loop

```python
class AutoReplyFilter:
    """
    Filters out auto-replies before they enter the feedback pipeline.
    Auto-replies (vacation, OOO, sabbatical, delivery failures) should
    not be classified as negative engagement.
    """
    def is_auto_reply(self, reply: Reply) -> bool:
        """
        Checks: X-Auto-Response-Suppress header,
        Precedence: auto_reply field,
        body contains 'sabbatical' | 'out of office' | 'vacation' | 'not reading email'
        """
        pass

class FeedbackLoop:
    """
    Processes replies to outbound emails:
    - Classifies sentiment
    - Updates researcher ranking
    - Updates template performance metrics
    """
    def poll_inbox(self) -> list[Reply]:
        """IMAP poll research@yourdomain.com for new replies."""
        pass

    def classify_reply(self, reply: Reply) -> Classification:
        """
        Classify: positive / negative / need-info / OOO / spam
        """
        pass

    def update_ranking(self, reply: Reply) -> None:
        """
        Positive → boost researcher's rank weight
        Negative / no-reply → decay
        """
        pass

    @dataclass
    class Reply:
        from_address: str
        subject: str
        body: str
        timestamp: datetime
        thread_id: str

    @dataclass
    class Classification:
        category: str  # "positive" | "negative" | "need_info" | "ooo" | "spam"
        confidence: float
        suggested_follow_up: bool
```

---

## 4. Data Model

### 4.1 Candidate (Discovery Output)

```json
{
  "id": "A5012345678",
  "name": "Jane Smith",
  "email": "jane.smith@university.edu",
  "email_source": "openalex_metadata",
  "orcid": "0000-0001-2345-6789",
  "affiliation": "University of Example",
  "institution_domain": "university.edu",
  "ror_id": "https://ror.org/042nb2s44",
  "metrics": {
    "h_index": 42,
    "works_count": 187,
    "cited_by_count": 15420,
    "two_year_mean_citedness": 45.3
  },
  "topic_relevance": 0.89,
  "sources": ["openalex", "semanticscholar"],
  "discovered_at": "2026-07-21T04:00:00Z",
  "run_id": "20260721_040000"
}
```

### 4.2 Queue Item

```json
{
  "id": "q_abc123",
  "candidate_id": "A5012345678",
  "email": "jane.smith@university.edu",
  "enqueued_at": "2026-07-21T05:00:00Z",
  "priority": 5,
  "age_count": 0,
  "channel": "email",
  "template": "cold_outreach_v1",
  "status": "pending",
  "attempts": 0,
  "last_error": null
}
```

### 4.3 Outbound Log

```json
{
  "message_id": "msg_abc123",
  "channel": "smtp",
  "to": "jane.smith@university.edu",
  "template": "cold_outreach_v1",
  "sent_at": "2026-07-21T10:00:00Z",
  "delivered": true,
  "bounced": false,
  "complaint": false,
  "reply_received": null,
  "follow_up_sent": false,
  "follow_up_count": 0
}
```

### 4.4 Storage Layout

```
~/.hermes/data/research_candidates/
├── 20260721_040000/
│   ├── candidates.json        # Phase 2 output
│   ├── enriched.json          # Phase 3 output
│   └── metadata.json          # Run metadata (APIs used, errors, counts)
├── 20260722_040000/
│   └── ...

~/.hermes/data/outbound_queue/
├── pending/
│   └── q_abc123.json          # Awaiting review gate approval
├── approved/
│   └── q_abc124.json          # Approved, queued for sending
└── processed/
    └── q_abc125.json          # Sent, logged

~/.hermes/data/outbound_log/
├── sent.json                  # Append-only log of all sent messages
├── bounces.json               # Bounce events
└── complaints.json            # Spam complaint events (triggers circuit breaker)

~/.hermes/data/feedback/
├── replies.json               # All classified replies
├── ranking_weights.json       # Per-researcher ranking adjustments
└── template_metrics.json      # Per-template response rates
```

---

## 5. Security & Compliance

### 5.1 Email Deliverability Security

| Control | Mechanism | Enforcement |
|---------|-----------|-------------|
| SPF | TXT record: `v=spf1 mx -all` | Recipient server rejects if IP not in list |
| DKIM | Cryptographic signing of all outgoing mail | Recipient verifies signature against DNS |
| DMARC | `p=quarantine` policy, reporting to postmaster | Recipient follows policy on failures |
| MTA-STS | HTTPS-hosted policy file | Recipient enforces TLS for MX connections |
| PTR/rDNS | VPS provider PTR record | Recipient checks reverse DNS match |
| Circuit breaker | Auto-stop on bounce >3% or any complaint | Programmatic, hard-coded, not configurable |

### 5.2 TOS Compliance

Each platform channel enforces its own compliance rules:

| Platform | Disclosure Requirement | Rate Limit | Attribution |
|----------|----------------------|------------|-------------|
| Stack Exchange | Post body must state "This query was composed with AI assistance for research purposes" | 1 post/site/day | Clear attribution, no impersonation |
| Reddit | Post must indicate automated assistance | 1 post/sub/day | Account dedicated to this purpose |
| ResearchGate | Manual review before posting (Playwright) | 1 post/day | Natural language, no disclose template |
| Email | Footer: "I'm a researcher investigating X. Reply to unsubscribe." | Warmup schedule | Easy opt-out, no follow-up after opt-out |

### 5.3 Privacy & GDPR

- **Data minimization:** Only store email, name, affiliation, and public metrics. No personal metadata beyond what's publicly available.
- **Retention:** Delete contact records after 90 days of no engagement. Archive logs after 1 year.
- **Right to erasure:** Unsubscribe = immediate deletion from active queue. Email to `privacy@yourdomain.com` triggers full deletion.
- **Legitimate interest basis:** Documented assessment that academic outreach (non-commercial, research-purposes) constitutes legitimate interest under GDPR Art. 6(1)(f).
- **Data portability:** User can export all contact data at any time via `research-tools export`.

### 5.4 Credential Management

All API keys, OAuth tokens, and SMTP passwords stored in SOPS-encrypted files:
- `~/.hermes/secrets/` — encrypted per-file
- Decrypted at runtime by `sops` or `systemd` `LoadCredential`
- No credentials in environment variables, git history, or log files
- Keys rotated every 90 days (or immediately on suspected compromise)

---

## 6. Deployment Architecture

### 6.1 Physical Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    Hetzner CX22 VPS                           │
│                    2 vCPU, 4GB RAM, 40GB SSD                  │
│                    Ports: 25/465/587/993/8080                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   NixOS (single flake)                    │ │
│  │  services.stalwart = { enable = true; ... }              │ │
│  │                                                           │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │  Stalwart     │  │  Python      │  │  Cron        │   │ │
│  │  │  (SMTP/IMAP)  │  │  Pipelines   │  │  Scheduler   │   │ │
│  │  │  ~180MB RAM   │  │  (discovery, │  │  (systemd    │   │ │
│  │  │               │  │   enrichment,│  │   timers)    │   │ │
│  │  │               │  │   outbound)  │  │              │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  │                                                           │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │  Data Store   │  │  Playwright  │  │  Hermes      │   │ │
│  │  │  (research_   │  │  (browser    │  │  Agent MCP   │   │ │
│  │  │   candidates, │  │   automation)│  │  (research   │   │ │
│  │  │   outbound_   │  │              │  │   tools)     │   │ │
│  │  │   log, etc.)  │  │              │  │              │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Resource Budget

| Service | RAM | CPU | Storage | Notes |
|---------|-----|-----|---------|-------|
| Stalwart | ~180MB | Negligible | ~100MB + email storage | Rust binary, single process |
| Cron pipelines | ~50MB per run | Burst | ~500MB for data | Python scripts, JSON I/O |
| Playwright | ~200MB | Burst | ~200MB | Headless Chromium |
| OS + system | ~500MB | — | ~5GB | NixOS minimal |
| **Total** | **~930MB** | **Low** | **~6GB** | Well within CX22 capacity |

### 6.3 Cron Schedule

All pipeline jobs write a `_COMPLETE` marker file before finishing. Downstream jobs check for this marker before reading data — this prevents race conditions when a run takes longer than the schedule offset.

```bash
# Discovery writes marker
touch /var/data/research_candidates/{run_id}/_COMPLETE

# Enrichment checks marker before reading
if [ ! -f /var/data/research_candidates/latest/_COMPLETE ]; then
    log "Discovery still running, skipping this cycle"
    exit 0
fi
```

| Job | Cron Expression | Duration | Notes |
|-----|----------------|----------|-------|
| `expert-discovery` | `0 */6 * * *` | ~5-15min | API calls are the bottleneck |
| `expert-enrich` | `30 */6 * * *` | ~5-10min | Offset 30min after discovery |
| `outbound-4a` | `0 */12 * * *` | ~5min | 10-20 emails, 1-2 forum posts |
| `outbound-4b` | `0 8 * * *` | ~10min | ResearchGate, daily |
| `digitization-sweep` | `0 10 * * *` | ~5-10min | Archive.org + form submission |
| `feedback-integrate` | `0 12 * * *` | ~5min | IMAP poll + classification |
| `response-monitor` | `0 */6 * * *` | ~2min | Quick inbox check |

---

## 7. Monitoring & Operations

### 7.1 Key Metrics

| Metric | Source | Warning | Critical | Response |
|--------|--------|---------|----------|----------|
| Bounce rate | Stalwart logs | >1% | >3% | Circuit breaker OPEN |
| Spam complaints | Stalwart + DMARC reports | ANY | ANY | Circuit breaker OPEN, alert user |
| Reply rate | Feedback loop | <5% | <2% | Review templates, check targeting |
| Discovery coverage | Discovery script | <50 candidates | <10 candidates | Check API keys, rate limits |
| Queue depth | Queue manager | >100 items | >500 items | Check backpressure, outbound health |
| Domain reputation | mail-tester.com | <8/10 | <5/10 | Pause outbound, investigate |
| API error rate | Per-API tracking | >5% | >20% | Alert, switch to fallback |

### 7.2 Alerting

All alerts delivered through Hermes cron notification system:
- **CRITICAL** (circuit breaker open, spam complaint) → immediate delivery to user
- **WARNING** (bounce rate rising, queue depth growing) → daily digest
- **INFO** (warmup milestone reached, review gate auto-promoted) → weekly summary

### 7.3 Warmup Progression

The IP warmup schedule is the single most important operational constraint. Rushing it permanently damages domain reputation.

| Week | Daily Volume | Circuit Breaker | Verification |
|------|-------------|-----------------|-------------|
| 1 | 2-5 | Auto-stop on bounce >3% or ANY complaint | Send to known-valid addresses only |
| 2 | 5-10 | Auto-stop on bounce >3% or ANY complaint | Include replies to self-domain |
| 3 | 10-20 | Auto-stop on bounce >3% | Include cross-domain replies |
| 4 | 20-50 | Auto-stop on bounce >3% | |
| 5-7 | 50-75 | Warning on bounce >3%, stop on >5% | |
| 8+ | 50-100 | Warning on bounce >3%, stop on >5% | Steady state |

---

## 8. Architecture Decision Records

### ADR-001: Stalwart over Mailcow

**Status:** Accepted
**Context:** Need a self-hosted email server on NixOS. Options: Stalwart, Mailcow, Mailu, manual Postfix+Dovecot stack.
**Decision:** Stalwart — Rust single binary, ~180MB RAM (vs Mailcow's 2-6GB), native NixOS module (`services.stalwart`), built-in JMAP, spam filtering, and TLS.
**Consequences:**
- + Minimal resource footprint, fits on CX22 alongside other services
- + Single config file, no Docker compose overhead
- + JMAP support enables modern programmatic email access
- - Smaller community than Mailcow (fewer tutorials)
- - Less mature (but production-stable for single-user mail)

### ADR-002: Pipeline Metapattern over Event-Driven

**Status:** Accepted
**Context:** The system has sequential stages (discover → enrich → outbound) that each depend on the previous stage's output. Could use event-driven (pub/sub) or pipeline (sequential stages).
**Decision:** Pipeline metapattern with explicit cron scheduling. Each stage writes structured JSON consumed by the next. No message broker needed.
**Consequences:**
- + Simple to reason about — each stage is a standalone script
- + Easy to debug — inspect JSON files at each stage boundary
- + No additional infrastructure (no Kafka, RabbitMQ, etc.)
- - Harder to add real-time processing (not needed here)
- - Cron-based = 6h minimum latency (acceptable for this use case)

### ADR-003: Plugin Pattern for Outbound Channels

**Status:** Accepted
**Context:** Multiple outbound channels (SMTP, Stack Exchange, Reddit, ResearchGate) with different APIs, auth methods, and rate limits. Need a uniform interface so the channel router can treat them interchangeably.
**Decision:** Plugin pattern — each channel implements `OutboundChannel` ABC with `send()`, `can_send()`, `remaining_quota()`. Channel registry maps intent to adapter.
**Consequences:**
- + Adding a new channel (e.g., LinkedIn, Twitter) requires only implementing the interface
- + Channel router logic is channel-agnostic
- + Rate limiting is per-channel, not centrally managed
- - Interface must be general enough to cover all channels (some methods may be no-ops)

### ADR-004: Circuit Breaker over Rate Limiting Alone

**Status:** Accepted
**Context:** Email reputation is the most constrained resource. Simple rate limiting (max N/day) is insufficient — a single spam complaint can destroy a domain's reputation permanently.
**Decision:** Hard circuit breaker — auto-stop ALL outbound on bounce >3% or any spam complaint. Reset requires manual intervention (with half-open test).
**Consequences:**
- + Protects domain reputation at the cost of delivery availability
- + Simple, unambiguous threshold (no false positives from spikes)
- + Manual reset ensures human reviews the situation before resuming
- - Aggressive: a single spam complaint from a misconfigured auto-responder could stop outbound
- - Mitigation: allowlist known-valid addresses in Phase 1 warmup

### ADR-005: OpenAlex + Semantic Scholar Dual API

**Status:** Accepted
**Context:** Need researcher discovery. Single API creates a single point of failure. Both APIs are free and complementary.
**Decision:** Use both OpenAlex and Semantic Scholar in parallel. Cross-reference and deduplicate results. OpenAlex as primary (broader coverage, includes email), Semantic Scholar as fallback + enrichment.
**Consequences:**
- + Redundancy — if one API is down, the other still produces results
- + Complementary data — OpenAlex has institutional metadata, Semantic Scholar has better citation graphs
- + Cross-referencing improves accuracy (confirm researcher across both sources)
- - Twice the API calls per run
- - Deduplication logic is non-trivial (name normalization, ORCID matching)

### ADR-006: Review Gate with Auto-Promotion

**Status:** Accepted
**Context:** Automated outbound without human oversight risks TOS violations, reputation damage, and poorly-worded messages. But requiring manual approval for every message forever is not scalable.
**Decision:** Implement a review gate (Phase 3.5) that requires human approval for all outbound messages initially. After 5 clean batches with 0 complaints, the gate auto-promotes to direct sending. The circuit breaker remains as a safety net.
**Consequences:**
- + Protects against early-stage mistakes during template tuning
- + Auto-promotion means the system becomes fully autonomous over time
- + Circuit breaker is a second layer of defense even after auto-promotion
- - Initial 5 batches require user attention (can be done in one session)
- - If auto-promotion triggers too early, reputation damage may occur before the circuit breaker catches it

---

## 9. Risk Register

| Risk | Impact | Likelihood | Mitigation | Contingency |
|------|--------|-----------|------------|-------------|
| Email reputation destroyed | Critical | Medium | Circuit breaker, warmup schedule, separate VPS | New domain + warmup from scratch (8 weeks) |
| TOS violation — account banned | High | Medium-High | Disclosure, human review gate, conservative rate limits | Alternative channel (email instead of forum) |
| OpenAlex API deprecated | High | Low | Semantic Scholar as fallback, local cache | Manual discovery via email |
| ResearchGate blocks automation | Medium | Medium | Human-like timing, CAPTCHA detection | Email fallback to same researcher |
| Hunter.io free tier insufficient | Medium | High | Pattern guessing + SMTP verify as primary | Upgrade to paid tier only if needed |
| GDPR complaint | Medium | Low | Privacy notice, easy opt-out, retention limits, documented legitimate interest | Immediate deletion + legal review |
| LinkedIn API blocked (partner-only) | Medium | High | Investigated in Phase 4B, documented as likely blocked | ResearchGate + email covers same use case |
| Domain not yet registered | Medium | N/A | Phase 1 must be completed before any sending | N/A — gating dependency |
| Warmup interrupted | Medium | Medium | Warmup script is idempotent — can resume from any point | If paused >7 days, restart from 2 weeks earlier |

---

## 10. Integration Points

### 10.1 Hermes Cron Integration

```
┌─────────────────────────────────────────────────┐
│              Hermes Cron Scheduler                │
│  (systemd timers or hermes cronjob create)        │
├─────────────────────────────────────────────────┤
│                                                   │
│  expert-discovery (every 6h)                       │
│    → write_file ~/.hermes/data/research_candidates/ │
│                                                   │
│  expert-enrich (6h+30m)                           │
│    → read_file + enrich + write_file               │
│                                                   │
│  outbound-4a (every 12h)                          │
│    → read enriched → send via SMTP/SE/Reddit       │
│    → circuit breaker check before each batch       │
│                                                   │
│  outbound-4b (daily)                              │
│    → Playwright automation → send via RG           │
│                                                   │
│  digitization-sweep (daily)                       │
│    → API checks + form submissions                 │
│                                                   │
│  feedback-integrate (daily)                       │
│    → IMAP poll → classify → update ranking         │
│                                                   │
│  response-monitor (every 6h)                      │
│    → Quick inbox check → log responses             │
└─────────────────────────────────────────────────┘
```

### 10.2 MCP Gateway Integration

The system can be invoked ad-hoc via Hermes MCP tools:

```python
# Ad-hoc expert query
research_tools.find_experts(topic="quantum error correction", limit=10)
# → Returns ranked list with contact info

# Ad-hoc outreach
research_tools.contact_expert(
    researcher_id="A5012345678",
    query="What is the current state of topological qubit research?",
    channel="email"
)
# → Sends query, returns message_id

# Ad-hoc digitization check
research_tools.check_digitization(
    work_title="A History of the Calculus of Variations",
    author="Goldstine",
    year=1980
)
# → Returns digitization status + request if needed
```

---

## 11. Evolution Path

### Phase 1 (Foundation) → Phase 6 (Full Orchestration)

The system is designed to evolve incrementally:

| Stage | Capabilities | Max Autonomy |
|-------|-------------|-------------|
| Foundation | Email infra, warmup, circuit breaker | 0% (no pipeline) |
| Discovery | Phase 2 + 3 — can find experts | 30% (output requires human action) |
| Review Gate | Phase 3.5 — human reviews all outbound | 50% (generates candidates, human sends) |
| Full Auto | Phase 4A + 4B + 6 — sends and tracks | 85% (circuit breaker can override) |
| Self-Improving | Phase 6.5 + 6.6 — feedback loop + A/B testing | 95% (human monitors dashboards) |

### Future Extensions

| Extension | Prerequisite | Effort |
|-----------|-------------|--------|
| LinkedIn channel adapter | Phase 4B LinkedIn research | Low if API available, Medium if Playwright |
| CleverX API integration | Paid subscription ($100/mo) | Low (REST API) |
| Guidepoint MCP transcript library | Phase 4A stable | Low (existing MCP integration) |
| Knowledge base of successful contacts | Phase 6 stable | Low (fact_store integration) |
| Multi-language outreach | Phase 6 stable | Medium (translation layer) |
| Automated conference paper discovery | Phase 2 stable | Medium (new API integration) |

---

## 12. Glossary

| Term | Definition |
|------|------------|
| Stalwart | Rust-based all-in-one mail server (SMTP, IMAP, JMAP) |
| OpenAlex | Open catalog of 321M+ scholarly works, with author/institution metadata |
| Semantic Scholar | AI-powered research tool with 200M+ papers, citation graphs |
| ORCID | Persistent digital identifier for researchers, with public profile data |
| Hunter.io | Email finding and verification API |
| ROR ID | Research Organization Registry — persistent identifier for institutions |
| Circuit breaker | Safety mechanism that stops outbound on delivery quality degradation |
| Review gate | Human approval step before messages are sent; auto-promotes after N clean batches |
| Backpressure | Mechanism that prevents upstream stages from producing when downstream is full |
| JMAP | Modern email protocol (RFC 8621) — JSON-based, replaces IMAP for programmatic access |

---

*Created 2026-07-21. Associated implementation plan: `Implementation_Plan.md` (90 tasks, 8 phases). Associated brief: `Expert_Research_Outreach_Brief.md` (vision, channel analysis, cost estimates).*