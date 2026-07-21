# Expert Research & Outreach — Test Plan

**Version:** 1.0
**Created:** 2026-07-21
**Methodology:** TDD (Red-Green-Refactor) for all unit tests + multi-specialist adversarial review
**Test framework:** pytest (Python) with pytest-mock, pytest-asyncio, responses (HTTP mocking)
**Adversarial layer:** Code Red-Team (Inquisitor) — 8-point output contract, FMEA, attack-path analysis
**Coverage target:** 85%+ line coverage, 100% of component interfaces, all risk register scenarios

---

## 1. Test Strategy Overview

### 1.1 Test Pyramid

```
         ╱╲
        ╱ E2E ╲             3-5 full pipeline tests (dry-run mode)
       ╱────────╲
      ╱Integration╲         15-20 cross-component tests
     ╱──────────────╲
    ╱   Unit Tests    ╲     60-80 isolated component tests
   ╱────────────────────╲
  ╱   Adversarial FMEA   ╲   20-30 failure-mode tests
 ╱──────────────────────────╲
╱  Property-based + Invariant ╲  10-15 generative tests
╱──────────────────────────────╲
```

### 1.2 Testing Principles

| Principle | Application |
|-----------|-------------|
| **TDD (Red-Green-Refactor)** | Every unit test is written BEFORE the implementation code. Watched fail. Watched pass. |
| **Mock external APIs** | OpenAlex, Semantic Scholar, ORCID, Hunter.io, Stalwart SMTP, Stack Exchange, Reddit, Archive.org, HathiTrust — all mocked in unit tests. Real API calls only in integration tests. |
| **Test behavior, not implementation** | Tests verify contracts and outcomes, not internal method calls. Refactoring should not break tests. |
| **One behavior per test** | Single assertion per test. "and" in the test name = split it. |
| **Edge cases first** | Empty responses, API timeouts, rate limits, malformed data, network failures — tested before happy path. |
| **Dry-run safety** | All outbound channels have a `--dry-run` mode that logs instead of sends. Integration tests run in dry-run mode by default. |
| **Adversarial complement** | After the TDD test suite passes, run the Code Red-Team (Inquisitor) for FMEA, attack-path analysis, and failure-mode validation. |

### 1.3 Test Directory Structure

```
expert-research-outreach/
├── src/
│   ├── discovery/
│   │   ├── openalex_client.py
│   │   ├── semanticscholar_client.py
│   │   ├── orcid_client.py
│   │   └── discovery_engine.py
│   ├── enrichment/
│   │   ├── hunter_client.py
│   │   ├── pattern_guesser.py
│   │   ├── smtp_verifier.py
│   │   └── enrichment_pipeline.py
│   ├── queue/
│   │   └── priority_queue.py
│   ├── outbound/
│   │   ├── base_channel.py
│   │   ├── smtp_channel.py
│   │   ├── se_channel.py
│   │   ├── reddit_channel.py
│   │   ├── rg_channel.py
│   │   └── channel_router.py
│   ├── feedback/
│   │   ├── auto_reply_filter.py
│   │   ├── reply_classifier.py
│   │   └── feedback_loop.py
│   ├── safety/
│   │   ├── circuit_breaker.py
│   │   └── rate_limiter.py
│   └── digitization/
│       ├── archive_client.py
│       ├── hathitrust_client.py
│       └── digitization_pipeline.py
├── tests/
│   ├── conftest.py              # Shared fixtures, mocks, factories
│   ├── fixtures/
│   │   ├── api_responses/        # JSON response fixtures
│   │   │   ├── openalex_author.json
│   │   │   ├── semanticscholar_paper.json
│   │   │   ├── orcid_record.json
│   │   │   ├── hunter_email.json
│   │   │   └── archive_metadata.json
│   │   └── sample_data.py        # Factory functions for test data
│   ├── unit/
│   │   ├── test_discovery.py
│   │   ├── test_enrichment.py
│   │   ├── test_queue.py
│   │   ├── test_outbound_channels.py
│   │   ├── test_circuit_breaker.py
│   │   ├── test_rate_limiter.py
│   │   ├── test_feedback.py
│   │   ├── test_auto_reply_filter.py
│   │   └── test_digitization.py
│   ├── integration/
│   │   ├── test_discovery_to_enrichment.py
│   │   ├── test_enrichment_to_queue.py
│   │   ├── test_queue_to_outbound.py
│   │   ├── test_outbound_to_feedback.py
│   │   └── test_circuit_breaker_integration.py
│   ├── e2e/
│   │   └── test_full_pipeline.py
│   ├── adversarial/
│   │   ├── test_fmea.py
│   │   ├── test_api_failures.py
│   │   └── test_queue_edge_cases.py
│   └── property/
│       └── test_invariants.py
├── pytest.ini
└── TEST_PLAN.md                  # This file
```

---

## 2. Test Environment

### 2.1 Environment Matrix

| Environment | Purpose | APIs | Notes |
|-------------|---------|------|-------|
| **Unit** | Component isolation | All mocked | Fast, no network, CI-friendly |
| **Integration** | Cross-component with real fixtures | Mocked + recorded responses | VCR.py for API recording |
| **E2E** | Full pipeline | Stalwart test instance + recorded API responses | Dry-run mode for outbound |
| **Adversarial** | Failure modes, fuzzing | All mocked | Chaos engineering patterns |
| **Staging** | Pre-production | Real Stalwart + test accounts on SE/Reddit/RG | Manual approval gate |

### 2.2 Fixtures & Mocks

```python
# conftest.py — shared fixtures

@pytest.fixture
def mock_openalex(requests_mock):
    """Mock OpenAlex /authors endpoint with sample responses."""
    with open("tests/fixtures/api_responses/openalex_author.json") as f:
        data = json.load(f)
    requests_mock.get(
        "https://api.openalex.org/authors?search=quantum+computing",
        json=data
    )
    return requests_mock

@pytest.fixture
def mock_semanticscholar(requests_mock):
    """Mock Semantic Scholar /paper/search endpoint."""
    with open("tests/fixtures/api_responses/semanticscholar_paper.json") as f:
        data = json.load(f)
    requests_mock.get(
        "https://api.semanticscholar.org/graph/v1/paper/search",
        json=data
    )
    return requests_mock

@pytest.fixture
def sample_candidate() -> DiscoveryEngine.Candidate:
    return DiscoveryEngine.Candidate(
        id="A5012345678",
        name="Jane Smith",
        email="jane.smith@university.edu",
        orcid="0000-0001-2345-6789",
        affiliation="University of Example",
        institution_domain="university.edu",
        h_index=42,
        works_count=187,
        cited_by_count=15420,
        two_year_mean=45.3,
        topic_relevance=0.89,
        source="both"
    )

@pytest.fixture
def sample_enriched() -> EnrichmentPipeline.EnrichedCandidate:
    return EnrichmentPipeline.EnrichedCandidate(
        id="A5012345678",
        name="Jane Smith",
        email="jane.smith@university.edu",
        confidence=0.95,
        verification_method="metadata",
        alt_emails=["jane.s@university.edu"]
    )

@pytest.fixture
def empty_queue() -> PriorityQueue:
    return PriorityQueue(storage_path=tempfile.mkdtemp())

@pytest.fixture
def closed_circuit_breaker() -> CircuitBreaker:
    return CircuitBreaker(
        bounce_threshold=0.03,
        cooldown_seconds=300
    )
```

### 2.3 Test Configuration

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
markers =
    unit: Unit tests (no network, no dependencies)
    integration: Integration tests (cross-component)
    e2e: End-to-end tests (full pipeline)
    adversarial: Failure-mode and fuzz tests
    property: Property-based and invariant tests
    slow: Tests that take >5 seconds
    network: Tests that make real network calls
filterwarnings =
    ignore::DeprecationWarning
addopts = -v --tb=short --strict-markers
```

---

## 3. Unit Tests

### 3.1 Discovery Engine

**File:** `tests/unit/test_discovery.py`

| Test ID | Test Name | Behavior | Mocks | Edge Cases |
|---------|-----------|----------|-------|------------|
| D-001 | `test_discover_returns_ranked_candidates` | Given topic, returns ranked list sorted by h-index | OpenAlex, Semantic Scholar | Empty results, single result |
| D-002 | `test_discover_merges_duplicate_authors` | Same author from both APIs → merged, not duplicated | Both APIs returning same author | Name variant matching (J. Smith vs Jane Smith) |
| D-003 | `test_discover_handles_openalex_timeout` | OpenAlex timeout → Semantic Scholar fallback only | OpenAlex raises Timeout | Both APIs timeout |
| D-004 | `test_discover_handles_api_rate_limit` | 429 response → backoff, log warning, resume next cycle | OpenAlex returns 429 | Semantic Scholar also 429 |
| D-005 | `test_discover_filters_by_works_count` | `filter=works_count:>10` applied correctly | OpenAlex | Edge value 10, 11, 0 |
| D-006 | `test_discover_ranks_by_citation_velocity` | 2yr_mean_citedness used as secondary sort key | Both APIs | Equal h-index, differing velocity |
| D-007 | `test_discover_extracts_email_from_metadata` | Email field present in API response → extracted | OpenAlex with email field | No email field, malformed email |
| D-008 | `test_discover_maps_ror_to_domain` | ROR ID → institution domain | OpenAlex with ror_id | Unknown ROR, missing ROR |
| D-009 | `test_discover_handles_malformed_response` | API returns non-JSON | Both APIs return 200 with HTML | Empty body, truncated JSON |
| D-010 | `test_discover_writes_run_metadata` | Run metadata file created with API status, error counts | Both APIs | Partial failure (one API up, one down) |

### 3.2 Enrichment Pipeline

**File:** `tests/unit/test_enrichment.py`

| Test ID | Test Name | Behavior | Mocks | Edge Cases |
|---------|-----------|----------|-------|------------|
| E-001 | `test_enrich_priority_metadata_over_orcid` | Metadata email chosen over ORCID when both available | Hunter.io, ORCID | Both present, same address |
| E-002 | `test_enrich_priority_orcid_over_hunter` | ORCID email chosen over Hunter.io | Hunter.io, ORCID | Confidence 0.90 vs 0.85 |
| E-003 | `test_enrich_pattern_guess_fallback` | No metadata/ORCID/Hunter → pattern guess | All return None | Multiple patterns for same domain |
| E-004 | `test_enrich_smtp_verify_disabled_during_warmup` | SMTP verify skipped before week 8 | None | Verify called in week 8+ |
| E-005 | `test_enrich_pattern_guess_formats` | Correct patterns: first.last, flast, lastf, first_last | None | Hyphenated names, middle initials |
| E-006 | `test_enrich_handles_invalid_email_format` | Malformed email → filtered out, not enqueued | Hunter.io returns invalid | Empty string, missing @ |
| E-007 | `test_enrich_hunter_rate_limit` | Hunter.io 429 → skip, continue with next candidate | Hunter.io returns 429 | All subsequent candidates use pattern guess |
| E-008 | `test_enrich_empty_candidate_list` | Empty input → empty output, no errors | None | None |
| E-009 | `test_enrich_confidence_threshold` | Confidence below 0.4 → excluded from output | All methods | Boundary: 0.39, 0.40, 0.41 |

### 3.3 Priority Queue

**File:** `tests/unit/test_queue.py`

| Test ID | Test Name | Behavior | Edge Cases |
|---------|-----------|----------|------------|
| Q-001 | `test_enqueue_dequeue_fifo` | Items dequeued in enqueue order by default | Single item, 1000 items |
| Q-002 | `test_priority_ordering` | Higher priority items dequeued first | Same priority, mixed priorities |
| Q-003 | `test_age_items_boosts_priority` | Items age 1 point per 24h, boosting priority over time | Age overflow (priority > 10) |
| Q-004 | `test_dedup_by_email` | Same email → second enqueue returns False | Same email, different case |
| Q-005 | `test_backpressure_level_empty` | Empty queue → 0.0 | None |
| Q-006 | `test_backpressure_level_full` | Queue at capacity → 1.0 | None |
| Q-007 | `test_backpressure_level_partial` | Queue at 50% → 0.5 | Boundaries: 0.49, 0.50, 0.51 |
| Q-008 | `test_purge_expired_items` | Items older than max_age → removed | Boundary: exactly max_age |
| Q-009 | `test_retry_failed_item` | Failed item retried (increments attempts, preserves priority) | Max retries exceeded |
| Q-010 | `test_get_queue_depth_by_channel` | `get_queue_depth("smtp")` returns count for that channel | Unknown channel, empty channel |
| Q-011 | `test_queue_persistence` | Queue state survives script restart (JSON file) | Corrupted JSON file |

### 3.4 Outbound Channel Adapters

**File:** `tests/unit/test_outbound_channels.py`

| Test ID | Test Name | Behavior | Mocks | Edge Cases |
|---------|-----------|----------|-------|------------|
| O-001 | `test_smtp_channel_send` | Email sent via smtplib, TLS 587 | Stalwart SMTP server | Connection refused, auth failure |
| O-002 | `test_smtp_channel_rate_limit` | `remaining_quota()` returns correct count | RateLimiter mock | Zero quota, max quota |
| O-003 | `test_smtp_channel_attribution_footer` | Footer appended to all outgoing messages | None | Empty body, very long body |
| O-004 | `test_se_channel_post_question` | POST /questions/add with OAuth token | Stack Exchange API | Token expired, invalid site |
| O-005 | `test_se_channel_site_selection` | Topic → site mapping correct | None | Unknown topic → default |
| O-006 | `test_se_channel_disclosure_appended` | AI disclosure included in post body | None | TOS violation → skip site |
| O-007 | `test_reddit_channel_submit` | `subreddit.submit()` called with correct args | PRAW mock | Subreddit doesn't exist |
| O-008 | `test_reddit_channel_attribution` | Post indicates automated assistance | PRAW mock | None |
| O-009 | `test_rg_channel_playwright` | Playwright navigates to Q&A page, fills form | Playwright mock | Page not found, CAPTCHA detected |
| O-010 | `test_rg_channel_captcha_pauses` | CAPTCHA detected → pause, alert user, don't attempt bypass | Playwright CAPTCHA element | False positive (CAPTCHA-like element) |
| O-011 | `test_channel_router_smtp` | Email-type query → SMTPChannel | Channel registry | None |
| O-012 | `test_channel_router_se` | Broad question → SEChannel | Channel registry | None |
| O-013 | `test_channel_router_fallback` | Unrecognized query → SMTPChannel (default) | Channel registry | None |
| O-014 | `test_all_channels_dry_run` | Dry-run mode logs instead of sends | All channels | None |

### 3.5 Circuit Breaker

**File:** `tests/unit/test_circuit_breaker.py`

| Test ID | Test Name | Behavior | Edge Cases |
|---------|-----------|----------|------------|
| C-001 | `test_initial_state_closed` | New circuit breaker starts CLOSED | None |
| C-002 | `test_is_allowed_closed` | CLOSED → `is_allowed()` returns True | None |
| C-003 | `test_bounce_under_threshold` | Bounce rate <= 3% → remains CLOSED | Boundary: 2.9%, 3.0%, 3.1% |
| C-004 | `test_bounce_over_threshold_opens` | Bounce rate > 3% → OPEN state | Precision: 3.01% |
| C-005 | `test_spam_complaint_opens_immediately` | ANY spam complaint → OPEN immediately | Multiple complaints |
| C-006 | `test_is_allowed_open` | OPEN → `is_allowed()` returns False | None |
| C-007 | `test_half_open_after_cooldown` | After cooldown, auto-transitions to HALF_OPEN | Cooldown boundary |
| C-008 | `test_half_open_test_email_success` | Test email succeeds → CLOSED | None |
| C-009 | `test_half_open_test_email_fails` | Test email fails → OPEN again | Multiple failures |
| C-010 | `test_record_bounce_accumulates` | Multiple bounces tracked correctly | 100 bounces, 0 bounces |
| C-011 | `test_record_delivery_accumulates` | Successful deliveries tracked correctly | None |
| C-012 | `test_reset_clears_state` | Manual reset clears counters, returns to CLOSED | None |

### 3.6 Rate Limiter

**File:** `tests/unit/test_rate_limiter.py`

| Test ID | Test Name | Behavior | Edge Cases |
|---------|-----------|----------|------------|
| R-001 | `test_initial_quota_full` | New rate limiter returns full quota | None |
| R-002 | `test_consume_success` | `consume("smtp", 1)` returns True, decrements quota | None |
| R-003 | `test_consume_over_quota` | `consume()` when quota exhausted → False | Boundary: exactly at quota |
| R-004 | `test_quota_resets_after_window` | Quota resets after time window | Sub-second timing |
| R-005 | `test_warmup_stage_week_1` | `warmup_stage()` returns 1 | Starting from scratch |
| R-006 | `test_warmup_stage_week_8` | `warmup_stage()` returns 8+ | None |
| R-007 | `test_per_channel_quotas` | Different channels have independent quotas | 5 channels, 1 channel |
| R-008 | `test_remaining_quota_accuracy` | `remaining_quota()` matches consumed count | None |

### 3.7 Feedback Loop

**File:** `tests/unit/test_feedback.py`

| Test ID | Test Name | Behavior | Mocks | Edge Cases |
|---------|-----------|----------|-------|------------|
| F-001 | `test_classify_positive_reply` | Reply with "thank you, interesting question" → POSITIVE | None | Emoji-only, multi-language |
| F-002 | `test_classify_negative_reply` | Reply with "not interested" → NEGATIVE | None | Passive-aggressive, polite decline |
| F-003 | `test_classify_need_info` | Reply with "can you clarify what you mean by X" → NEED_INFO | None | Multiple questions |
| F-004 | `test_classify_ooo` | Reply with "I'm on sabbatical until March" → OOO | None | Date parsing, auto-reply headers |
| F-005 | `test_classify_spam` | Reply with "BUY NOW!!!" → SPAM | None | Phishing URLs |
| F-006 | `test_positive_updates_ranking` | POSITIVE → researcher's ranking weight increased | None | Already high weight cap |
| F-007 | `test_negative_decays_ranking` | NEGATIVE → researcher's ranking weight decreased | None | Weight floor (minimum) |
| F-008 | `test_follow_up_triggered` | No reply after N days → follow-up queued | None | Boundary: N-1, N, N+1 days |
| F-009 | `test_follow_up_not_triggered_after_reply` | Reply received → no follow-up | None | Reply received at exactly N days |

### 3.8 Auto-Reply Filter

**File:** `tests/unit/test_auto_reply_filter.py`

| Test ID | Test Name | Behavior | Edge Cases |
|---------|-----------|----------|------------|
| AR-001 | `test_detects_vacation_auto_reply` | "I'm out of the office" → True | Different languages |
| AR-002 | `test_detects_sabbatical_auto_reply` | "I'm on sabbatical until" → True | Date range included |
| AR-003 | `test_detects_delivery_failure` | "Delivery Status Notification (Failure)" → True | Bounce format variation |
| AR-004 | `test_detects_x_auto_header` | X-Auto-Response-Suppress header → True | Missing header |
| AR-005 | `test_passes_through_normal_reply` | "That's an interesting question" → False | Contains auto-reply keyword in context |
| AR-006 | `test_handles_empty_body` | Empty body → False | None |
| AR-007 | `test_handles_unicode` | Unicode body with auto-reply keywords → True | CJK, RTL, emoji |

### 3.9 Digitization Pipeline

**File:** `tests/unit/test_digitization.py`

| Test ID | Test Name | Behavior | Mocks | Edge Cases |
|---------|-----------|----------|-------|------------|
| DG-001 | `test_archive_org_finds_digitized` | Item already digitized → returns URL | Archive.org API | Multiple versions |
| DG-002 | `test_archive_org_not_found` | Item not digitized → returns None | Archive.org API | 404 vs 403 vs 500 |
| DG-003 | `test_hathitrust_check` | Item in partner collection → returns URL | HathiTrust API | Authentication required |
| DG-004 | `test_finding_aid_detector` | URL matches finding-aid pattern → returns institution | None | Multiple patterns, no match |
| DG-005 | `test_holding_institution_matcher` | Work metadata → likely holding institution | None | Multiple matches, no match |
| DG-006 | `test_digitization_email_generated` | Email template populated with correct fields | None | Missing fields, all fields |

---

## 4. Integration Tests

### 4.1 Discovery → Enrichment

**File:** `tests/integration/test_discovery_to_enrichment.py`

| Test ID | Test Name | What It Verifies |
|---------|-----------|------------------|
| DE-001 | `test_discovery_to_enrichment_full_flow` | 10 candidates discovered → enriched with verified emails → confidence scores assigned |
| DE-002 | `test_discovery_to_enrichment_partial_emails` | 10 candidates, 4 with metadata emails, 3 with ORCID, 3 pattern-only → correct confidence per source |
| DE-003 | `test_discovery_to_enrichment_all_fail` | All API calls fail → empty output, error logged |
| DE-004 | `test_discovery_to_enrichment_data_format` | Output JSON schema matches expected format |

### 4.2 Enrichment → Queue

**File:** `tests/integration/test_enrichment_to_queue.py`

| Test ID | Test Name | What It Verifies |
|---------|-----------|------------------|
| EQ-001 | `test_enriched_candidates_enqueued` | Enriched candidates → queue items with correct priority |
| EQ-002 | `test_enriched_duplicates_deduped` | Same email across two runs → second enqueue ignored |
| EQ-003 | `test_enriched_empty_skipped` | Candidates with no email → not enqueued |

### 4.3 Queue → Outbound

**File:** `tests/integration/test_queue_to_outbound.py`

| Test ID | Test Name | What It Verifies |
|---------|-----------|------------------|
| QO-001 | `test_queue_to_outbound_4a` | Queue items → SMTP/SE/Reddit outbound (dry-run) |
| QO-002 | `test_queue_to_outbound_4b` | Queue items → ResearchGate outbound (dry-run) |
| QO-003 | `test_queue_respects_rate_limits` | Rate limiter stops outbound at quota, resumes after reset |
| QO-004 | `test_queue_to_outbound_all_channels` | Channel router selects correct channel per query type |

### 4.4 Outbound → Feedback

**File:** `tests/integration/test_outbound_to_feedback.py`

| Test ID | Test Name | What It Verifies |
|---------|-----------|------------------|
| OF-001 | `test_outbound_log_updates_on_reply` | Sent message tracked → reply received → log updated |
| OF-002 | `test_auto_reply_filtered_out` | Auto-reply received → filtered, not classified as negative |
| OF-003 | `test_positive_reply_updates_ranking` | Positive reply → ranking weight of that researcher increases |
| OF-004 | `test_follow_up_triggered_on_no_reply` | No reply after N days → follow-up queued |

### 4.5 Circuit Breaker Integration

**File:** `tests/integration/test_circuit_breaker_integration.py`

| Test ID | Test Name | What It Verifies |
|---------|-----------|------------------|
| CB-001 | `test_circuit_breaker_stops_outbound` | Bounce rate >3% → circuit breaker OPEN → all outbound channels blocked |
| CB-002 | `test_circuit_breaker_spam_complaint` | Any spam complaint → circuit breaker OPEN → immediate stop |
| CB-003 | `test_circuit_breaker_auto_reset` | After cooldown → HALF_OPEN → test email succeeds → CLOSED |
| CB-004 | `test_circuit_breaker_manual_reset` | Manual reset → counters cleared → CLOSED |
| CB-005 | `test_circuit_breaker_does_not_affect_review_gate` | Circuit breaker OPEN → review gate still accepts new items (queued, not sent) |

---

## 5. End-to-End Tests

### 5.1 Full Pipeline (Dry-Run)

**File:** `tests/e2e/test_full_pipeline.py`

| Test ID | Test Name | What It Verifies |
|---------|-----------|------------------|
| FP-001 | `test_full_pipeline_topic_to_queued` | "quantum error correction" → discovered 10 candidates → enriched 7 with emails → all 7 queued → 0 sent (review gate) |
| FP-002 | `test_full_pipeline_with_review_approval` | Topic → pipeline → user approves 3 → 3 sent (dry-run) → 3 logged |
| FP-003 | `test_full_pipeline_with_reply` | Topic → pipeline → approved → sent → mock reply received → classified as positive → ranking updated |
| FP-004 | `test_full_pipeline_circuit_breaker_engagement` | Pipeline running → simulated bounce spike → circuit breaker OPEN → pipeline paused → alert logged |
| FP-005 | `test_full_pipeline_digitization` | Work title → Archive.org check (not found) → digitization email queued |

### 5.2 Dry-Run Mode

All E2E tests run in `--dry-run` mode by default:

```bash
# Run full pipeline without sending anything
python -m src.pipeline.run --dry-run --topic "quantum error correction"

# Verify: all outbound channels logged, nothing sent
grep "DRY_RUN" /var/data/outbound_log/sent.json
```

---

## 6. Adversarial & Red-Team Tests

### 6.1 FMEA (Failure Mode Effects Analysis)

**File:** `tests/adversarial/test_fmea.py`

For each component, systematically test all failure modes from the risk register:

| Test ID | Component | Failure Mode | Injection | Expected Result |
|---------|-----------|-------------|-----------|-----------------|
| FM-001 | Discovery | OpenAlex returns 500 | HTTP 500 mock | Fallback to Semantic Scholar, warning logged |
| FM-002 | Discovery | Both APIs return 500 | Both HTTP 500 | Skip cycle, alert |
| FM-003 | Discovery | OpenAlex returns stale data | 1-year-old data | Accepted with warning |
| FM-004 | Enrichment | Hunter.io returns 402 (quota) | HTTP 402 mock | Skip Hunter, use pattern guess |
| FM-005 | Enrichment | Pattern guesser produces 0 emails | All patterns fail | No candidates enriched, log warning |
| FM-006 | Queue | Queue file corrupted | Write garbage to file | Rebuild from outbound_log, warning |
| FM-007 | Queue | Disk full | Simulate ENOSPC | Graceful error, alert |
| FM-008 | Outbound | SMTP connection refused | Mock connection refused | Retry with backoff, max 3 attempts |
| FM-009 | Outbound | SE OAuth token expired | Mock 401 response | Refresh token, retry |
| FM-010 | Outbound | Reddit rate limited | Mock 429 response | Skip cycle, log |
| FM-011 | Circuit Breaker | Bounce counter overflow | 10,000 bounces | Cap at MAX_INT, no overflow |
| FM-012 | Feedback | IMAP connection fails | Mock connection error | Skip cycle, log warning |
| FM-013 | Feedback | Reply classifier returns low confidence | All <0.5 confidence | Flag for manual review |
| FM-014 | Digitization | Archive.org API changes response format | Unexpected JSON structure | Graceful parse error, log |

### 6.2 API Failure Scenarios

**File:** `tests/adversarial/test_api_failures.py`

| Test ID | Scenario | Injection | Expected |
|---------|----------|-----------|----------|
| AF-001 | OpenAlex rate limit (429) | 10 rapid requests | Backoff, resume next cycle |
| AF-002 | Semantic Scholar empty response | `{"data": []}` | Empty candidates, log warning |
| AF-003 | ORCID timeout | 30s TCP timeout | Skip, continue with other sources |
| AF-004 | Hunter.io malformed response | Non-JSON 200 | Error, skip Hunter |
| AF-005 | Stack Exchange API version change | Unexpected response fields | Fail gracefully, log |
| AF-006 | All external APIs down simultaneously | All return 503 | All pipelines pause, CRITICAL alert |

### 6.3 Queue Edge Cases

**File:** `tests/adversarial/test_queue_edge_cases.py`

| Test ID | Scenario | Expected |
|---------|----------|----------|
| QC-001 | 10,000 items enqueued simultaneously | All processed, no OOM |
| QC-002 | Item with priority overflow (>10) after aging | Capped at 10 |
| QC-003 | Dedup with 1,000 identical emails | 1 item enqueued, 999 rejected |
| QC-004 | Queue file simultaneously read/written | File lock acquired, no corruption |
| QC-005 | Backpressure at 0.8 → discovery stops producing | Discovery skips next cycle |

---

## 7. Property-Based & Invariant Tests

**File:** `tests/property/test_invariants.py`

Using Hypothesis or similar property-based testing:

| Test ID | Invariant | Input Space | Test |
|---------|-----------|-------------|------|
| PB-001 | Circuit breaker state machine | All possible (bounce, delivery, complaint, reset) sequences | From any state, the circuit breaker eventually returns to CLOSED after sufficient positive signals |
| PB-002 | Queue FIFO within priority | Any mix of enqueue/dequeue operations | Items with same priority are dequeued in enqueue order |
| PB-003 | Rate limiter monotonicity | Any consume + reset sequence | `remaining_quota()` never increases except after reset |
| PB-004 | Dedup idempotency | Any sequence of enqueue calls | Enqueuing the same email N times results in exactly 1 item |
| PB-005 | Confidence score bounds | All possible enrichment inputs | All confidence scores are in [0.0, 1.0] |
| PB-006 | Age can't decrease priority | Any aging schedule | `age_items()` never decreases an item's priority |

---

## 8. Security Tests

### 8.1 Credential Management

| Test ID | Test Name | What It Verifies |
|---------|-----------|------------------|
| S-001 | `test_no_credentials_in_code` | grep for `api_key`, `secret`, `password`, `token` in `src/` — zero matches |
| S-002 | `test_no_credentials_in_logs` | Log output redacts credentials (replaces with `****`) |
| S-003 | `test_sops_encrypted_files` | All secret files under `~/.hermes/secrets/` are SOPS-encrypted |
| S-004 | `test_env_file_not_committed` | `.env` in `.gitignore`, not tracked by git |

### 8.2 TOS Compliance Enforcement

| Test ID | Test Name | What It Verifies |
|---------|-----------|------------------|
| T-001 | `test_all_outbound_has_attribution` | Every channel adapter appends attribution/disclosure |
| T-002 | `test_site_policy_lookup_enforced` | Sites with `ban` policy → channel skips, logs reason |
| T-003 | `test_review_gate_requires_approval` | Messages in review queue → NOT sent until approved |
| T-004 | `test_opt_out_removes_from_queue` | Unsubscribe link clicked → email removed from all queues |

### 8.3 GDPR Compliance

| Test ID | Test Name | What It Verifies |
|---------|-----------|------------------|
| G-001 | `test_data_retention_enforced` | Contacts with no engagement for 90 days → deleted |
| G-002 | `test_erasure_request_processed` | Email to privacy@ → full deletion within 24h |
| G-003 | `test_data_portability_export` | `research-tools export` produces valid JSON |
| G-004 | `test_privacy_notice_in_footer` | All outbound emails include privacy notice + opt-out link |

---

## 9. Test Execution & CI

### 9.1 Test Commands

```bash
# Run all tests
pytest

# Run by category
pytest -m unit
pytest -m integration
pytest -m e2e
pytest -m adversarial
pytest -m property

# Run specific test file
pytest tests/unit/test_circuit_breaker.py -v

# Run with coverage
pytest --cov=src --cov-report=term-missing

# Run in dry-run mode (safety check)
pytest -m "not network" --dry-run

# Run adversarial tests specifically
pytest tests/adversarial/ -v --tb=long
```

### 9.2 CI Pipeline (GitHub Actions)

```yaml
name: Test Suite
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      - run: pip install uv && uv sync
      - run: pytest -m "unit or integration" --cov=src --cov-report=term-missing
      - run: pytest -m adversarial -v --tb=long
        if: github.ref == 'refs/heads/main'
      - run: pytest -m e2e --dry-run
        if: github.ref == 'refs/heads/main'
```

### 9.3 Coverage Gates

| Gate | Threshold | Enforcement |
|------|-----------|-------------|
| Line coverage | >= 85% | CI failure if below |
| Branch coverage | >= 75% | CI warning if below |
| Adversarial tests | 0 failures | CI failure if any fail |
| Security tests | 0 failures | CI failure if any fail |
| Property tests | 0 failures | CI failure if any fail |

---

## 10. Test Data Management

### 10.1 Fixture JSON Files

Fixed sample responses for all external APIs, stored in `tests/fixtures/api_responses/`:

| File | Source | Fields |
|------|--------|--------|
| `openalex_author.json` | OpenAlex `/authors` | 10 authors with varying h-index, affiliation, email presence |
| `semanticscholar_paper.json` | Semantic Scholar `/paper/search` | 15 papers with authors, citation counts |
| `orcid_record.json` | ORCID `/record` | 5 profiles with mixed email visibility |
| `hunter_email.json` | Hunter.io `/email-finder` | 5 verified, 3 unverified, 2 not found |
| `archive_metadata.json` | Archive.org metadata | 2 digitized, 2 not found, 1 restricted |

### 10.2 Factory Functions

```python
# tests/fixtures/sample_data.py

def make_candidate(overrides: dict = None) -> dict:
    """Generate a valid candidate dict with randomizable fields."""
    base = {
        "id": f"A{random.randint(1000000, 9999999)}",
        "name": fake.name(),
        "email": fake.email(),
        "orcid": None,
        "affiliation": fake.company(),
        "h_index": random.randint(5, 80),
        "works_count": random.randint(10, 500),
        "cited_by_count": random.randint(100, 50000),
        "two_year_mean": round(random.uniform(0, 100), 1),
        "topic_relevance": round(random.uniform(0, 1), 2),
        "source": random.choice(["openalex", "semanticscholar", "both"])
    }
    if overrides:
        base.update(overrides)
    return base

def make_queue_item(overrides: dict = None) -> dict:
    """Generate a valid queue item."""
    return {
        "id": f"q_{uuid.uuid4().hex[:8]}",
        "candidate_id": f"A{random.randint(1000000, 9999999)}",
        "email": fake.email(),
        "enqueued_at": datetime.now().isoformat(),
        "priority": random.randint(1, 10),
        "age_count": 0,
        "channel": random.choice(["email", "stackexchange", "reddit"]),
        "template": "cold_outreach_v1",
        "status": "pending",
        "attempts": 0,
        "last_error": None,
        **(overrides or {})
    }
```

---

## 11. Development Order (TDD Sequence)

Tests must be written and passing in this order, following the TDD Red-Green-Refactor cycle:

| Order | Component | Tests | Depends On |
|-------|-----------|-------|------------|
| 1 | Circuit Breaker | C-001 to C-012 | None |
| 2 | Rate Limiter | R-001 to R-008 | None |
| 3 | Auto-Reply Filter | AR-001 to AR-007 | None |
| 4 | Discovery Engine | D-001 to D-010 | None |
| 5 | Enrichment Pipeline | E-001 to E-009 | None |
| 6 | Priority Queue | Q-001 to Q-011 | None |
| 7 | Outbound Channels | O-001 to O-014 | Rate Limiter |
| 8 | Feedback Loop | F-001 to F-009 | Auto-Reply Filter |
| 9 | Digitization Pipeline | DG-001 to DG-006 | None |
| 10 | Integration: Discovery→Enrichment | DE-001 to DE-004 | 4, 5 |
| 11 | Integration: Enrichment→Queue | EQ-001 to EQ-003 | 5, 6 |
| 12 | Integration: Queue→Outbound | QO-001 to QO-004 | 6, 7 |
| 13 | Integration: Outbound→Feedback | OF-001 to OF-004 | 7, 8 |
| 14 | Integration: Circuit Breaker | CB-001 to CB-005 | 1, 7 |
| 15 | Adversarial FMEA | FM-001 to FM-014 | All unit tests |
| 16 | API Failures | AF-001 to AF-006 | All unit tests |
| 17 | Queue Edge Cases | QC-001 to QC-005 | 6 |
| 18 | Property-Based | PB-001 to PB-006 | All unit tests |
| 19 | Security & Compliance | S-001 to G-004 | Integration tests |
| 20 | E2E Full Pipeline | FP-001 to FP-005 | All integration tests |

---

## 12. Test Success Criteria

The test plan is complete when:

- [ ] 80+ unit tests passing (all components)
- [ ] 15+ integration tests passing (all pipeline boundaries)
- [ ] 5 E2E tests passing (dry-run mode)
- [ ] 14 adversarial FMEA scenarios passing (all failure modes)
- [ ] 6 property-based invariants verified (generative)
- [ ] 10+ security/GDPR tests passing (compliance)
- [ ] Line coverage >= 85%
- [ ] All tests pass in CI (GitHub Actions)
- [ ] Dry-run mode verified as safe (no outbound messages sent)
- [ ] Circuit breaker verified as effective (stops outbound on threshold breach)
- [ ] Review gate verified as enforced (no messages bypass approval)

---

## 13. Appendix: TDD Workflow for Each Component

For each component, follow this exact sequence:

```
1. Write test file with all test functions
2. Run tests → RED (all fail — no implementation)
3. Write minimal implementation to pass first test
4. Run that specific test → GREEN
5. REFACTOR (clean up, keep tests green)
6. Write next test → RED
7. Extend implementation → GREEN
8. REFACTOR
9. Repeat until all tests for that component are green
10. Run full test suite → no regressions
11. Commit: "feat: implement <component> with TDD"
```

See `test-driven-development` skill for the full TDD methodology.

---

*Created 2026-07-21. Associated design doc: `ARCHITECTURE.md`. Associated implementation plan: `Implementation_Plan.md`.*