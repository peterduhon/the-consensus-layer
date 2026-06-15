# 06: SLA and Uptime Guarantees

**The Consensus Layer: Chapter 1 — AdTech Translation**
*A shared language for product teams translating business logic into technical infrastructure.*

---

## The Business Problem

A Service Level Agreement is a contractual commitment: the platform will be available, responsive, and accurate for a defined percentage of time within a defined measurement period, and if it is not, the client is entitled to a defined remedy. SLAs exist because downtime in adtech is not an inconvenience; it is a revenue event for both parties. When a data pipeline fails and a client's campaign cannot deliver, the impressions that did not serve do not come back. When a reporting platform goes dark and a client cannot reconcile their spend, the billing dispute that follows is a renewal conversation in disguise.

The gap between engineering's mental model of availability (the system is up and running) and the commercial model (the system is available, correct, and within SLA bounds) is where some of the most expensive misunderstandings in adtech platform management occur. Startup-era engineering teams that built systems for technical, self-sufficient clients; often DSPs running their own infrastructure and capable of absorbing occasional data gaps; are now serving enterprise clients who manage $10M quarterly media contracts and expect the same operational standards they receive from their cloud providers and financial data vendors.

"99.9% uptime" sounds like a strong commitment. For a data pipeline serving a $10M quarterly contract, it means 8.7 hours of allowable downtime per year. Whether that is acceptable depends entirely on when those 8.7 hours occur and what they cost the client.

---

## What Engineers Think

"99.9% is fine. We're already close to that. It's mostly stable."

Engineers who have never worked under a formal SLA often conflate "the system is running" with "the system is meeting its commitment." They track uptime as a percentage of total time rather than as a budget of allowable downtime. This is not a skill gap; it is a contract literacy gap. They have never had to translate a percentage into a dollar amount, a remediation clause, or a renewal conversation.

The commercial definition of availability differs from the engineering definition in three specific ways.

**Measurement window matters:** A system that is down for 4 hours on a Tuesday afternoon may be 99.98% available that month. A system that is down for 4 hours during a major live sporting event, a product launch, or an upfront buying window is a $500K billing dispute regardless of the monthly percentage. Engineers who track availability as a rolling average miss the business impact of availability at specific moments.

**Data correctness is part of the SLA:** A system that is technically "up" but delivering incorrect impression counts, misattributed conversions, or delayed log exports is not meeting its commercial obligation even if the uptime percentage is 100%. This requires separating availability SLAs (is the API responding?) from data quality SLAs (are the numbers correct?). The monitoring infrastructure for data quality is different: it requires reconciliation queries, beacon event completeness checks, and third-party verification deltas rather than simple health checks.

**The cost asymmetry is invisible without the contract:** A 0.1% error rate on a system processing 10 million impressions per day is 10,000 errors daily. At $25 CPM, that is $250 in direct billing error; if the error is random and uncorrelated, this is within standard IAB discrepancy tolerance (5%) and is unlikely to surface as a dispute. But if the 0.1% error is systematic; all from a single client, all on a single day, all in the same direction; the $250 becomes a $10,000 billing dispute that surfaces in a quarterly business review and anchors the renewal conversation. The SLA must distinguish between random error variance and systematic error concentration. Without a contract that specifies both error rate tolerance and error distribution requirements, systematic errors are invisible to engineering until they become commercial crises.

Engineers from startup backgrounds often operated in environments where the client base was technically sophisticated enough to absorb errors and where informal relationships handled disputes. At enterprise scale, with legal teams reviewing contracts and procurement processes requiring SLA language, "mostly stable" is not a commitment.

---

## What the Business Actually Needs

### The SLA Vocabulary: Four Metrics That Belong in Every Contract

| Metric | Definition | Typical Target | Measurement Period | What Breaks Without It |
|--------|------------|---------------|-------------------|------------------------|
| **Uptime / Availability** | Percentage of time the system is operational and accepting requests | 99.5% to 99.9% | Monthly rolling | Clients cannot serve campaigns; delivery gaps are unrecoverable |
| **Log delivery latency** | Time from impression event to log record availability for client access | p99 under 4 hours for batch; under 200ms for real-time | Per-delivery window | Client reconciliation pipelines fail; billing disputes follow |
| **Impression count accuracy** | Delta between platform-reported impressions and client-side or third-party verification | Within 5% (IAB standard) | Per-campaign | Billing discrepancies; advertiser withholds payment; renewal at risk |
| **Data completeness** | Percentage of expected records present in each log delivery | 99.5% or higher | Per-delivery | Missing records create gaps in attribution, frequency enforcement, and client reporting |

### The Error Budget: Making the Tradeoff Explicit

An error budget is the operational consequence of an SLA commitment: if the SLA target is 99.9% uptime, the error budget for the month is 0.1% of total time; 43.8 minutes per month; 8.7 hours per year. The error budget makes the abstract percentage concrete for engineering teams.

**Why error budgets change engineering behavior:** When an engineering team understands that the monthly error budget is 43 minutes and they have already consumed 30 minutes in the first two weeks of the month, release decisions change. A risky deployment that might cause 15 minutes of degraded performance gets pushed to the next release window rather than going out on a Friday afternoon. The budget makes the cost of risk visible before the risk is taken.

**The product manager's role in error budget governance:** The error budget is a shared resource between product and engineering. Product managers control two inputs: the pace of feature releases (more releases = more deployment risk = more error budget consumption) and the SLA tier committed to clients (higher SLA = smaller error budget = less room for risky releases). Neither team can optimize its own behavior without visibility into how the other team's decisions affect the budget.

```python
# Error budget calculator: the arithmetic that makes SLA percentages real
# Use this to translate abstract uptime percentages into concrete time allowances
# before committing to a client-facing SLA tier.

from dataclasses import dataclass
from typing import Optional

@dataclass
class ErrorBudget:
    """
    Computes the error budget for a given SLA tier and measurement period.
    The error budget is the maximum allowable downtime before the SLA is breached.
    """
    sla_pct: float          # e.g. 99.9 for 99.9% uptime
    period_days: int        # measurement period in days (typically 30)

    @property
    def allowed_downtime_minutes(self) -> float:
        total_minutes = self.period_days * 24 * 60
        return total_minutes * (1 - self.sla_pct / 100)

    @property
    def allowed_downtime_hours(self) -> float:
        return self.allowed_downtime_minutes / 60

    @property
    def error_rate_pct(self) -> float:
        return 100 - self.sla_pct

    def budget_remaining(self, downtime_consumed_minutes: float) -> dict:
        remaining = self.allowed_downtime_minutes - downtime_consumed_minutes
        pct_consumed = (downtime_consumed_minutes / self.allowed_downtime_minutes) * 100
        return {
            'allowed_minutes':    round(self.allowed_downtime_minutes, 1),
            'consumed_minutes':   round(downtime_consumed_minutes, 1),
            'remaining_minutes':  round(remaining, 1),
            'pct_consumed':       round(pct_consumed, 1),
            'status':             'BREACHED' if remaining < 0 else (
                                  'AT_RISK'  if pct_consumed > 80 else 'HEALTHY'
            )
        }


# SLA tier comparison: what each tier actually means in time
SLA_TIERS = {
    '99.0%': ErrorBudget(sla_pct=99.0, period_days=30),
    '99.5%': ErrorBudget(sla_pct=99.5, period_days=30),
    '99.9%': ErrorBudget(sla_pct=99.9, period_days=30),
    '99.95%': ErrorBudget(sla_pct=99.95, period_days=30),
    '99.99%': ErrorBudget(sla_pct=99.99, period_days=30),
}

def compare_sla_tiers() -> None:
    """
    Print a comparison of SLA tiers and their error budgets.
    Use this in product discussions before committing to a client SLA tier.
    The goal: make sure the number in the contract is a number the
    engineering team can actually meet, based on current baseline performance.
    """
    print(f"{'SLA Tier':<10} {'Budget (min/mo)':<20} {'Budget (hr/mo)':<18} {'Budget (hr/yr)'}")
    print("-" * 70)
    for tier, budget in SLA_TIERS.items():
        annual_hours = budget.allowed_downtime_hours * 12
        print(
            f"{tier:<10} "
            f"{budget.allowed_downtime_minutes:<20.1f} "
            f"{budget.allowed_downtime_hours:<18.2f} "
            f"{annual_hours:.2f}"
        )

# Output:
# SLA Tier   Budget (min/mo)      Budget (hr/mo)     Budget (hr/yr)
# -----------------------------------------------------------------------
# 99.0%      432.0                7.20               86.40
# 99.5%      216.0                3.60               43.20
# 99.9%      43.2                 0.72                8.64
# 99.95%     21.6                 0.36                4.32
# 99.99%     4.3                  0.07                0.88
```

**The product decision:** Never commit to a client SLA tier without first establishing a 30-day baseline of actual system performance. A platform that averages 99.3% uptime over the prior 30 days cannot credibly commit to 99.9% without the engineering work to close the gap. The sequence that works: instrument first; establish internal targets based on the baseline; meet those targets consistently for four to six weeks; then introduce the SLA commitment in new contracts.

### Circuit Breakers and Graceful Degradation: The Engineering Response to SLA Pressure

When an SLA commitment exists, the engineering response to failure changes. Without an SLA, a degraded state (slow but running) is acceptable; the system is still "up." With an SLA, a degraded state that falls below the response time threshold counts against the error budget. This creates pressure for two engineering patterns that startup-era teams often skip.

**Circuit breakers:** A circuit breaker monitors a downstream dependency (a database, an external API, a data vendor) and opens (trips) when the dependency begins failing consistently, preventing the calling service from continuing to make requests that will fail and accumulate latency. When the circuit is open, the service fails fast rather than waiting for timeouts; preserving response time for the requests that can succeed.

**Graceful degradation:** A system that degrades gracefully under load or dependency failure serves reduced functionality rather than failing completely. For an adtech data pipeline, graceful degradation might mean: serving cached impression counts from the last successful batch rather than returning an error when the live query fails; or serving audience segments from the prior day's model run rather than erroring on real-time inference when the ML scoring service is degraded.

```python
# Circuit breaker implementation: illustrative pattern
# Production circuit breaker libraries: resilience4j (Java), pybreaker (Python),
# hystrix (Netflix, now mostly legacy). Use a library; don't implement from scratch.
# This illustrates the state machine logic that underlies any circuit breaker.

import time
from enum import Enum
from typing import Callable, Any

class CircuitState(Enum):
    CLOSED   = "closed"    # Normal operation; requests pass through
    OPEN     = "open"      # Failure threshold exceeded; requests fail fast
    HALF_OPEN = "half_open" # Testing recovery; one probe request allowed

class CircuitBreaker:
    """
    Wraps a callable (API call, DB query, external service request) with
    circuit breaker logic. Prevents cascading failures when a dependency degrades.

    State transitions:
      CLOSED  → OPEN:      failure_threshold consecutive failures
      OPEN    → HALF_OPEN: after recovery_timeout seconds
      HALF_OPEN → CLOSED:  one successful probe request
      HALF_OPEN → OPEN:    probe request fails; reset timeout

    For adtech data pipelines: wrap calls to external data vendors, identity
    graph providers, and real-time scoring services. When those dependencies
    degrade, fail fast and serve cached or degraded responses rather than
    consuming error budget on timeouts.
    """

    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout_seconds: int = 60,
        name: str = "unnamed"
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout_seconds
        self.name = name

        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._last_failure_time: Optional[float] = None

    @property
    def state(self) -> CircuitState:
        if self._state == CircuitState.OPEN:
            if time.time() - self._last_failure_time > self.recovery_timeout:
                self._state = CircuitState.HALF_OPEN
        return self._state

    def call(self, fn: Callable, *args, fallback: Any = None, **kwargs) -> Any:
        """
        Execute fn with circuit breaker protection.
        If the circuit is OPEN, return fallback immediately (fail fast).
        If the circuit is HALF_OPEN, allow one probe; reopen on failure.
        """
        if self.state == CircuitState.OPEN:
            # Fail fast: do not make the request; preserve error budget
            return fallback

        try:
            result = fn(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            if fallback is not None:
                return fallback  # Graceful degradation: serve cached/default
            raise

    def _on_success(self) -> None:
        self._failure_count = 0
        self._state = CircuitState.CLOSED

    def _on_failure(self) -> None:
        self._failure_count += 1
        self._last_failure_time = time.time()
        if self._failure_count >= self.failure_threshold:
            self._state = CircuitState.OPEN
```

### SQL: SLA Monitoring Dashboard Queries

```sql
-- ─────────────────────────────────────────────────────────────────────────────
-- QUERY A: Uptime and Availability Baseline
-- Establishes the 30-day rolling uptime baseline before any SLA commitment.
-- "Instrument before you commit" is the first rule of SLA introduction.
-- Written for Snowflake syntax.
-- BigQuery: replace DATEADD with DATE_SUB; DATEDIFF syntax differs
-- Redshift: DATEADD compatible; window functions supported
-- ─────────────────────────────────────────────────────────────────────────────

WITH health_events AS (
    SELECT
        event_timestamp,
        service_name,          -- 'log_pipeline', 'reporting_api', 'segment_service'
        event_type,            -- 'healthy', 'degraded', 'down'
        error_code,
        response_time_ms,
        event_date
    FROM service_health_log
    WHERE event_date >= DATEADD(day, -30, CURRENT_DATE)
),

availability_by_service AS (
    SELECT
        service_name,
        COUNT(*)                                                    AS total_checks,
        SUM(CASE WHEN event_type = 'healthy'  THEN 1 ELSE 0 END)  AS healthy_checks,
        SUM(CASE WHEN event_type = 'degraded' THEN 1 ELSE 0 END)  AS degraded_checks,
        SUM(CASE WHEN event_type = 'down'     THEN 1 ELSE 0 END)  AS down_checks,
        ROUND(
            SUM(CASE WHEN event_type = 'healthy' THEN 1 ELSE 0 END)
            * 100.0 / NULLIF(COUNT(*), 0),
            3
        )                                                           AS uptime_pct,
        ROUND(AVG(CASE WHEN event_type = 'healthy' THEN response_time_ms END), 1)
                                                                    AS avg_response_ms_healthy,
        ROUND(PERCENTILE_CONT(0.99) WITHIN GROUP
              (ORDER BY response_time_ms), 1)                       AS p99_response_ms
    FROM health_events
    GROUP BY service_name
)

SELECT
    service_name,
    uptime_pct,
    -- Error budget consumption against common SLA tiers
    ROUND((100 - uptime_pct) / (100 - 99.5) * 100, 1)              AS pct_of_99_5_budget_used,
    ROUND((100 - uptime_pct) / (100 - 99.9) * 100, 1)              AS pct_of_99_9_budget_used,
    avg_response_ms_healthy,
    p99_response_ms,
    -- Flag services not yet ready for external SLA commitment
    CASE
        WHEN uptime_pct < 99.0 THEN 'NOT_SLA_READY: below 99% baseline'
        WHEN uptime_pct < 99.5 THEN 'MONITOR: between 99.0% and 99.5%'
        WHEN p99_response_ms > 500 THEN 'LATENCY_RISK: p99 response exceeds 500ms'
        ELSE 'SLA_ELIGIBLE: baseline supports external commitment'
    END                                                              AS sla_readiness

FROM availability_by_service
ORDER BY uptime_pct ASC;  -- Surface lowest-performing services first


-- ─────────────────────────────────────────────────────────────────────────────
-- QUERY B: Log Delivery Latency SLA Tracking
-- For data platforms: availability of the API is necessary but not sufficient.
-- Log delivery latency is often the SLA metric clients actually enforce.
-- ─────────────────────────────────────────────────────────────────────────────

SELECT
    event_date,
    client_id,
    log_type,                  -- 'impression_log', 'completion_log', 'attribution_log'
    COUNT(*)                                                         AS deliveries,
    ROUND(AVG(delivery_latency_minutes), 1)                         AS avg_latency_min,
    ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP
          (ORDER BY delivery_latency_minutes), 1)                   AS p95_latency_min,
    ROUND(PERCENTILE_CONT(0.99) WITHIN GROUP
          (ORDER BY delivery_latency_minutes), 1)                   AS p99_latency_min,
    -- Flag deliveries that breach the 4-hour SLA window (240 minutes)
    SUM(CASE WHEN delivery_latency_minutes > 240 THEN 1 ELSE 0 END) AS sla_breaches,
    ROUND(
        SUM(CASE WHEN delivery_latency_minutes > 240 THEN 1 ELSE 0 END)
        * 100.0 / NULLIF(COUNT(*), 0),
        2
    )                                                                AS breach_rate_pct

FROM log_delivery_log
WHERE event_date >= DATEADD(day, -30, CURRENT_DATE)

GROUP BY event_date, client_id, log_type
ORDER BY breach_rate_pct DESC, event_date DESC;
```

**What these queries produce before any SLA conversation:** The `sla_readiness` field in Query A makes the go/no-go decision visible without requiring engineering to frame it as a business question. A service flagged `NOT_SLA_READY` cannot ethically be committed to a client SLA until the gap is closed. A service flagged `SLA_ELIGIBLE` has a baseline that supports the commitment. Query B surfaces the log delivery latency distribution; because for many CTV data platform clients, log delivery SLA is more operationally critical than API availability.

---

## The Three-Phase SLA Introduction Framework

Introducing SLA language into existing client relationships without the technical infrastructure to back the commitment creates legal and commercial liability. The framework below sequences the introduction to protect both the platform and the client relationship.

**Phase 1: Instrument before you commit (30 days)**
Deploy uptime monitoring, latency tracking, error rate dashboards, and log delivery latency measurement for all pipeline outputs before any SLA conversation happens internally or externally. You cannot commit to a target you cannot measure. Establish the 30-day baseline using Query A above. That baseline is the honest starting point; not an aspirational target.

**Phase 2: Internal targets before external commitments (days 31 to 60)**
Set internal team commitments based on the baseline data. These are not yet client-facing; they are the engineering team's commitments to itself. Review weekly in standup. Make the data visible to the whole team on a shared dashboard. This builds a culture of reliability before the obligation is external. An engineering team that has never tracked uptime as a first-class metric will not meet an external SLA target on the first try.

**Phase 3: New contracts first, existing clients second (days 61 to 90 and beyond)**
Once internal targets have been met consistently for four to six weeks, introduce SLA language into new client contracts. Start with a commitment the system can comfortably exceed. Then introduce SLA commitments to existing clients as a product enhancement rather than a contract amendment: "We are now providing formal uptime guarantees as part of our service." Frame it as a competitive differentiator; most adtech data platforms in CTV and DOOH do not offer formal SLAs.

---

## Failure Mode

**What breaks when a platform serving enterprise contracts has no SLAs:**

The absence of an SLA does not protect a platform from downtime consequences; it just makes the consequences unpredictable. When a client's campaign goes dark because a data pipeline fails and there is no SLA to define what the client is owed, the dispute is resolved by the commercial relationship rather than the contract. Commercial relationships that absorb too many unresolved incidents eventually reach a breaking point at renewal.

The specific failure mode that surfaces most often: a CTV or DOOH data platform built for technically sophisticated clients (early DSP partners, programmatic buyers who understood that data pipelines have latency and gaps) begins serving enterprise advertisers or agency holding companies with formal procurement processes. Those clients send SLA requirement questionnaires as part of onboarding. The platform has no SLA to provide. The onboarding stalls. The deal either closes on informal assurances that will not hold under the first incident, or it does not close at all.

The second failure mode: a platform that does have an SLA but set it based on aspirational targets rather than baseline measurement. A platform that committed to 99.9% uptime before establishing that its actual baseline was 99.2% has a 0.7 percentage point gap that will consume the error budget within days of each measurement period. Every breach requires a credit or a remediation conversation. Those conversations accumulate into a renewal risk.

The third failure mode is specific to data quality SLAs: a platform that tracks API availability but not impression count accuracy or log delivery latency. A client whose reconciliation pipeline fails because a log delivery was six hours late rather than the committed four hours does not distinguish between "the system was technically up" and "the system failed its SLA." Both produce a billing dispute.

This pattern has been observed across multiple CTV and adtech data platform transitions to enterprise scale. The gap is closed by instrumentation, baseline measurement, internal target culture, and the phased introduction sequence described above; not by committing to a percentage before the system can meet it.

**What to do when you breach:** The first SLA breach after introducing formal commitments will happen. The question is not whether but when; and the quality of the response determines whether the breach becomes a trust event or a trust-building event. The incident response sequence that works: acknowledge the breach within the SLA-defined notification window (typically 1 to 4 hours); quantify the impact in the client's terms (impressions undelivered, log delivery hours delayed, dollar value affected); deliver a root cause analysis within the contractually specified timeframe (typically 5 to 10 business days); and present the remediation plan with measurable milestones. Clients who receive a well-structured incident response typically renew; clients who receive silence or deflection often do not. The incident response process must be designed before the first breach, not invented during it. This is where platform engineering discipline; toggle-driven releases, circuit breakers, rollback mechanisms, and pre-defined escalation paths; becomes directly commercial: platforms that can recover quickly and communicate clearly during an incident spend less of the error budget per event.

---

## Translation Conversation: Product and Engineering

*How to explain SLA commitment consequences to an engineering team without SLA experience:*

"99.9% uptime sounds like we're almost always available. And we are; 99.9% is genuinely good. But here's what that number means in practice: we are allowed to be down for 43 minutes per month before we breach the contract. That's not 43 minutes spread evenly across the month. That's 43 minutes total. If we have a two-hour incident, we've breached the SLA for the month. If our client's campaign was running a major product launch during that two hours, the conversation about what we owe them starts immediately.

The SLA isn't there to punish us when things break. Things will always break. The SLA is there so that when things break, everyone knows what the rules are: how long the outage was, what it cost the client in undelivered impressions, and what remediation they're entitled to. Without an SLA, that conversation is about feelings and relationships. With an SLA, it's about math. Math is better.

The error budget is our tool for managing release risk. If we have 43 minutes of error budget for the month and we've already used 30, we don't ship a risky deployment on Friday afternoon. That's not a business decision overriding an engineering decision. That's a shared decision with a shared number driving it."

---

## Open-Source Implementation

**AdInsight: Reddit Ad Dashboard** (`github.com/peterduhon/adinsight`) demonstrates the same pattern of SQL-backed dashboard visualization applied to campaign performance metrics through a Streamlit interface. The SLA monitoring queries in this artifact are designed to run against any health event log schema. The next milestone for AdInsight extends that pattern to operational metrics: adding a `service_health_log` table to the SQLite schema, populating it with synthetic health check events across three service types (impression pipeline, reporting API, segment service), and implementing Query A above as a new dashboard tab that surfaces the `sla_readiness` classification in an interface context.

**Performance Optimization in AdTech Bidding** (`github.com/peterduhon/performance-optimization`) implements multithreaded bid simulation with execution time profiling and latency visualization. The p99 latency tracking in that repo is directly connected to the SLA monitoring queries in this artifact: pipeline latency at the p99 level is the metric that most often determines whether a data platform meets its log delivery SLA under load. The next milestone for that repo documents the circuit breaker pattern above as the latency defense mechanism under peak load scenarios.

The Consensus Layer's three-phase SLA introduction framework above emerged from direct observation at a CTV media company where engineering teams had built reliable systems for technically sophisticated early clients but had never worked under formal SLA commitments. The absence of SLAs at enterprise scale in that environment was not an oversight; it was a structural consequence of engineering teams that were never given the business context to understand what downtime costs a contract. The framework; instrument first, internal targets second, new contracts third, existing clients as a product enhancement; is designed to close that gap without creating contractual liability before the system can meet the commitment.

---

## Chapter 1: AdTech Translation — Full Series

| # | Concept | Technical Focus | Status |
|---|---------|-----------------|--------|
| 01 | [First-Price vs. Second-Price Auction](./01-first-price-second-price.md) | Bid shading; floor price optimization; log schema price fields | ✅ Published |
| 02 | [Viewability (MRC / IAS / MOAT)](./02-viewability.md) | Pixel firing; viewport detection; SSAI completion beacons; DOOH OTS | ✅ Published |
| 03 | [Frequency Capping](./03-frequency-capping.md) | Household graph deduplication; cross-device session management; DOOH screen-level vs audience-level | ✅ Published |
| 04 | [Attribution Window](./04-attribution-window.md) | Conversion pixel timing; post-click vs. post-view; consideration cycle configuration | ✅ Published |
| 05 | [Clean Rooms and PETs](./05-clean-rooms-pets.md) | Differential privacy; k-anonymity; query logging; privacy-preserving computation | ✅ Published |
| **06** | **SLA and Uptime Guarantees** | Error budgets; circuit breakers; graceful degradation; contract-level implications | ✅ Published |

**Chapter 1 is complete.** Chapters 2 through 4 extend The Consensus Layer beyond adtech translation into design intelligence, API-first product methodology, and enterprise product infrastructure. See the [repository README](../README.md) for the full project map.

---

## References and Further Reading

- Google SRE Book: "Service Level Objectives" chapter — the foundational treatment of error budgets, SLI/SLO/SLA distinctions, and the relationship between release velocity and reliability
- IAB Tech Lab: "Digital Advertising Measurement Guidelines" — impression count accuracy standards (5% tolerance), discrepancy resolution process, and data quality SLA norms
- IAB Tech Lab: "Programmatic Log-Level Data Standards" — log delivery latency expectations and schema completeness requirements for DSP and SSP clients
- Netflix Tech Blog: "Making the Netflix API More Resilient" — the circuit breaker pattern in production at scale; Hystrix origin story
- OpenDP: resilience4j documentation (Java) and pybreaker documentation (Python) — production circuit breaker libraries; do not implement from scratch
- Martin Fowler: "CircuitBreaker" pattern (martinfowler.com) — the canonical pattern description with state machine diagram
- The Consensus Layer Artifact 01: First-Price vs. Second-Price Auction — log delivery SLA and billing accuracy SLA share a schema dependency: `win_price`, `clearing_price`, and `floor_price` must be present and correct for both the auction reconciliation and the SLA measurement to function
- The Consensus Layer Artifact 04: Attribution Window — attribution SLAs (how long after the campaign ends must attribution data be available?) are a separate contract term from availability SLAs; the attribution schema in Artifact 04 must be present within the contracted attribution delivery window

---

*The Consensus Layer is a living reference built by Pete Duhon. Born in AdTech. Built for product teams translating business logic into technical infrastructure.*
*GitHub: github.com/peterduhon | Last updated: June 2026*
