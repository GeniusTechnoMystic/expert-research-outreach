# Design Review — Expert Research & Outreach Architecture

**Reviewer:** AION Meta-Strategist (multi-specialist)
**Date:** 2026-07-21
**Document reviewed:** `ARCHITECTURE.md` (835 lines, 12 sections)
**Supporting docs:** `Expert_Research_Outreach_Brief.md`, `Implementation_Plan.md`, `README.md`

---

## Summary

| Lens | Score | Bottom Line |
|------|-------|-------------|
| Architecture & Metapatterns | 8/10 | Pipeline + Plugin are correct. Orchestrator naming is aspirational. |
| Component Design | 7/10 | Missing RateLimiter, auto-reply filter. SMTP-verify is a bootstrapping risk. |
| Security & Compliance | 6/10 | SMTP verify triggers spam traps. Legitimate-interest assertion needs a real doc. |
| Operational Readiness | 8/10 | No backup for 8-week warmup state. Cron race condition on file I/O. |
| ADR Quality | 9/10 | Missing ADR for JSON-over-SQLite data store decision. |
| Risk Register | 7/10 | Missing: discovery model bias, reply classifier drift. |
| Completeness | 7/10 | Missing: testing strategy section, error taxonomy. |

**Overall:** A solid, implementable design with well-rationalized decisions. The gaps are operational (backups, race conditions) and one bootstrapping paradox (SMTP-verify triggering circuit breaker before warmup starts).

---

## Finding 1 — Orchestrator is a thin coordinator, not a conductor

**Severity:** Medium
**Section:** §2.1-2.2

The diagram shows an "Orchestrator Layer" with priority queue, backpressure controller, and circuit breaker. At implementation, the cron scheduler fires independent scripts. There's no central conductor that knows the state of the entire pipeline at runtime. The priority queue is a passive file store, not an active controller.

**Recommendation:** Rename to "Scheduling & Queue Layer" in the diagram, or commit to building a proper conductor service that can pause/resume stages, handle partial failures, and re-route around blocked channels.

---

## Finding 2 — Pipeline layering has cross-layer dependencies

**Severity:** Low
**Section:** §2.2

Phase 5 (Digitization) is shown running parallel with Phase 2, but tasks 5.5-5.8 (web forms, email sending) require Stalwart and the SMTP sender — components from Phase 1 and Phase 4A respectively. Only Phase 5.1-5.3 (archive API checks) are genuinely parallel.

**Recommendation:** Annotate the diagram: "Phase 5.1-5.3 parallel with Phase 2; Phase 5.5-5.8 serial behind Phase 4A."

---

## Finding 3 — Missing RateLimiter component

**Severity:** Medium
**Section:** §3 (missing)

Rate limiting is discussed in the channel registry table and ADR-004, but has no standalone component. The warmup schedule, per-channel rate limits, and daily caps are distributed across ad-hoc mechanisms.

**Recommendation:** Add a `RateLimiter` component:

```python
class RateLimiter:
    def remaining_quota(self, channel: str) -> int: ...
    def consume(self, channel: str, count: int = 1) -> bool: ...
    def warmup_stage(self) -> int:  # returns week number 1-8+
    def reset_at(self, channel: str) -> datetime: ...
```

---

## Finding 4 — FeedbackLoop misses auto-reply filter

**Severity:** Medium
**Section:** §3.6

Auto-replies (vacation, OOO, sabbatical, delivery failures) will be classified as "negative" and decay the researcher's ranking. These should be filtered before classification.

**Recommendation:** Add an `AutoReplyFilter` step before `classify_reply()`:

```python
class AutoReplyFilter:
    def is_auto_reply(self, reply: Reply) -> bool:
        # Checks: X-Auto-Response-Suppress, Precedence: auto_reply,
        # body contains "sabbatical" / "out of office" / "vacation"
```

---

## Finding 5 — QueueManager interface is thin

**Severity:** Low
**Section:** §3.4

The `PriorityQueue` has 5 stubs. For a "shared queue connecting all pipeline stages," operations like `get_queue_depth()`, `get_oldest_item_age()`, `retry_failed()`, and `purge_expired()` are needed.

**Recommendation:** Add operational methods to the interface:

```python
def get_queue_depth(self, channel: str) -> int: ...
def get_oldest_item_age(self) -> timedelta: ...
def retry_failed(self, item_id: str) -> None: ...
def purge_expired(self, max_age: timedelta) -> int: ...
```

---

## Finding 6 — SMTP verify can trigger spam traps (CRITICAL)

**Severity:** HIGH
**Section:** §3.2

The enrichment pipeline uses SMTP/MX verification (HELO check, no delivery). Some mail servers treat any connection from an unknown IP as a spam signal, especially at volume. This could trigger the circuit breaker before warmup begins — creating a bootstrapping paradox: enrichment triggers spam detection → circuit breaker opens → warmup never starts.

**Recommendation:** Three options (choose one):
1. Disable SMTP verify entirely (rely on Hunter.io + pattern confidence)
2. Run SMTP verify from a separate IP (second microVM)
3. Rate-limit SMTP verify to 5 checks/day during warmup period
Add a note: "SMTP verify is disabled by default. Enable manually once domain reputation stabilizes (week 8+)."

---

## Finding 7 — Review gate CLI has no authentication guard

**Severity:** Medium
**Section:** §3.5 (implied)

The user approves/rejects messages through a CLI. If a Hermes session is compromised (via Matrix bridge, cron task injection), the approval action could be used to push malicious content.

**Recommendation:** Add explicit guard: "Review gate approval is LOCAL ONLY. If remote approval is needed, add a confirmation challenge (re-type 'APPROVE')."

---

## Finding 8 — GDPR legitimate-interest assessment needs a living document

**Severity:** Low (for design phase)
**Section:** §5.3

The design asserts legitimate interest under GDPR Art. 6(1)(f) but has no actual assessment document. Before Phase 4A goes live, this must exist.

**Recommendation:** Add a Phase 6 task: "Write legitimate-interest assessment document covering: (1) purpose of processing, (2) necessity, (3) balance of interests, (4) less intrusive alternatives considered, (5) safeguards for data subjects."

---

## Finding 9 — Stack Exchange TOS varies by site

**Severity:** Medium
**Section:** §5.2

SE network-wide rules on AI-generated content vary by site. A blanket disclosure policy may miss site-specific bans or requirements.

**Recommendation:** Add site-level policy lookup:

```python
# Channel router checks before posting
SITE_AI_POLICY_CACHE = {
    "stackoverflow.com": "allow_with_disclosure",
    "academia.stackexchange.com": "allow_with_disclosure",
    "mathoverflow.net": "ban",  # example
}
```

---

## Finding 10 — No backup plan for warmup state (HIGH)

**Severity:** HIGH
**Section:** §4.4 / §7 (missing)

The data store is on a single VPS. If it's lost, 8 weeks of IP warmup, all discovery/enrichment history, and the feedback learning are gone. Warmup is not reproducible in less than 8 weeks.

**Recommendation:** Add backup strategy to §7:

```yaml
backup:
  schedule: daily at 02:00 UTC
  target: s3://backblaze-b2/expert-research-outreach/
  contents:
    - ~/.hermes/data/warmup_state.json
    - ~/.hermes/data/queue/
    - ~/.hermes/data/outbound_log/
  retention: 90 days
  restore:
    - Install new VPS
    - Restore latest backup tarball
    - Continue warmup from last checkpoint
```

---

## Finding 11 — Cron race condition on file I/O

**Severity:** Medium
**Section:** §6.3

Discovery (`0 */6 * * *`) and enrichment (`30 */6 * * *`) have a 30-minute offset. If a discovery run takes 45 minutes (API timeouts + retries), enrichment reads partial data.

**Recommendation:** Add lockfile marker pattern:

```bash
# Discovery writes a _COMPLETE marker when done
touch /var/data/research_candidates/{run_id}/_COMPLETE

# Enrichment checks for marker before reading
if [ ! -f /var/data/research_candidates/latest/_COMPLETE ]; then
    echo "Discovery still running, skipping this cycle"
    exit 0
fi
```

---

## Finding 12 — Warmup threshold relaxation is too aggressive

**Severity:** Low
**Section:** §7.3

Week 8 changes volume AND circuit breaker threshold simultaneously — two risk variables at once.

**Recommendation:** Keep circuit breaker at "stop on >3%" through week 12. Only relax after 4 more weeks of clean delivery.

---

## Finding 13 — Missing: discovery model bias

**Risk Register gap**

OpenAlex and Semantic Scholar over-index on English-language STEM research and well-funded institutions. Researchers in non-STEM fields at Global South universities may be invisible.

**Recommendation:** Add to risk register:

| Risk | Impact | Likelihood | Mitigation | Contingency |
|------|--------|-----------|------------|-------------|
| Discovery model bias — systematic coverage gaps | Medium | High | Add non-English sources (CNKI, SciELO). Document known gaps. | Manual discovery via domain-specific sources |

---

## Finding 14 — Missing: reply classifier drift

**Risk Register gap**

The AI-based reply classifier will degrade over time as language patterns shift.

**Recommendation:** Add to risk register:

| Risk | Impact | Likelihood | Mitigation | Contingency |
|------|--------|-----------|------------|-------------|
| Reply classifier accuracy drifts over time | Medium | Medium | Log classified sample weekly for manual review. Track confidence distribution. Alert on drop. | Retrain classifier on newly labeled data |

---

## Finding 15 — Missing: testing strategy

**Severity:** Medium
**Section:** All (missing)

The Implementation Plan has testing tasks but the architecture doc doesn't describe mocking strategy for external APIs, integration test environment, or circuit breaker testing.

**Recommendation:** Add a "Testing Strategy" section covering:
1. Unit tests: mock all external APIs, test each component in isolation
2. Integration tests: test full pipeline against a staging Stalwart instance
3. Circuit breaker tests: simulate bounce, spam complaint, verify state machine
4. Dry-run mode: log instead of send (from Implementation Plan task 6.16)

---

## Finding 16 — Missing ADR: JSON files over SQLite

**Severity:** Low

JSON flat files are chosen over SQLite but the decision isn't documented as an ADR. The trade-offs are real: JSON is human-readable and simple; SQLite is queryable and transactional.

**Recommendation:** Add ADR-007:

**Status:** Accepted
**Context:** Need to store pipeline state (candidates, queue items, logs). Options: JSON files, SQLite, PostgreSQL.
**Decision:** JSON files organized by run ID. No database server needed. Human-readable, git diff, grep-able.
**Consequences:**
- + Zero infrastructure, zero dependencies
- + Easy to inspect and debug
- + Git-friendly for small datasets
- - No query language — must use grep/jq/Python
- - No transactions — file writes must be atomic (write to tmp then rename)

---

## Action Priority

| Priority | Finding | Action |
|----------|---------|--------|
| 🔴 HIGH | F6 — SMTP verify triggers spam traps | Disable or rate-limit to 5/day |
| 🔴 HIGH | F10 — No backup for warmup state | Add daily backup to Backblaze B2 |
| 🟡 MEDIUM | F3 — Missing RateLimiter component | Add to §3 |
| 🟡 MEDIUM | F4 — Auto-reply filter missing | Add to FeedbackLoop |
| 🟡 MEDIUM | F9 — SE TOS varies by site | Add site-level policy lookup |
| 🟡 MEDIUM | F11 — Cron race condition | Add lockfile marker pattern |
| 🟡 MEDIUM | F15 — Testing strategy missing | Add §13 to architecture doc |
| 🟢 LOW | F1, F2, F5, F7, F8, F12, F13, F14, F16 | Decorative/long-term improvements |

---

*Reviewed 2026-07-21 by AION Meta-Strategist. Fixed: ARCHITECTURE.md updated for F3, F4, F6, F11 (see commit after this review).*