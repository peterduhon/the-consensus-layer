# 05: Clean Rooms and Privacy-Enhancing Technologies (PETs)

**The Consensus Layer: Chapter 1 — AdTech Translation**
*A shared language for product teams translating business logic into technical infrastructure.*

---

## The Business Problem

Two parties in the advertising ecosystem each hold data the other needs. A publisher knows who visits their platform, what content they consume, and how long they stay. An advertiser knows who has purchased their product, who is in their CRM, and who they want to reach next. Neither party is willing to hand their data to the other in raw form: the publisher's first-party audience data is a competitive asset; the advertiser's CRM is commercially sensitive; and both parties face regulatory constraints that prohibit raw data exchange under GDPR, CPRA, and increasingly under state-level privacy laws.

Clean rooms exist to answer the question both parties need answered (how much does my audience overlap with yours, and what does that tell us about where to spend advertising budget?) without either party exposing the raw data that makes the question possible to answer in the first place.

The business consequence of getting this wrong is not a compliance violation in isolation: it is the loss of the data collaboration that enables premium CPMs. Publishers who can demonstrate audience overlap with an advertiser's known customers command 2 to 5 times the CPM of equivalent untargeted inventory. That premium only exists when the measurement is trusted. Trust requires that the clean room is actually clean.

---

## What Engineers Think

"Clean room means encrypted. We hash the email addresses, neither party sees the other's raw data, we're done."

Engineers encountering clean room requirements for the first time often model the problem as a one-time data masking step: hash the PII, join the hashed values, report the overlap count. This model solves the raw data exposure problem at the point of the join. It does not solve the inference problem that emerges from the query outputs.

**The inference problem:** A COUNT query run against a clean room can leak information about individuals even when no raw records are returned. If an advertiser can submit arbitrary queries against a publisher's first-party data and receive exact counts, they can use differencing attacks to isolate individual records: submit one query for "users in zip code X who visited health content," then another for "users in zip code X who visited health content AND are female," compare the counts, and infer the gender of identifiable individuals in that cohort. This is not a theoretical attack; it is a documented vulnerability in naive clean room implementations.

**Why encryption alone does not address this:** SHA-256 hashing of email addresses before the join protects raw PII from crossing organizational boundaries. It does not protect the statistical properties of the data from leaking through query outputs. Two fundamentally different controls are needed: one for data at rest and in transit (hashing, tokenization), and one for query outputs (differential privacy, k-anonymity thresholds, query auditing).

Engineers who implement the first control without the second have built a system that is technically defensible in a security audit but commercially and legally vulnerable in a privacy audit.

---

## What the Business Actually Needs

### The Three Layers of a Clean Room

A production clean room is not a single technology; it is a stack of three distinct controls that operate at different points in the data collaboration lifecycle.

| Layer | What It Protects | How It Works | Where It Lives |
|-------|-----------------|--------------|----------------|
| **Data layer** | Raw PII from crossing organizational boundaries | SHA-256 hashing; tokenization; pseudonymization | ETL pipeline; data ingestion |
| **Query layer** | Individual-level inference from aggregate outputs | Differential privacy noise injection; k-anonymity thresholds; query auditing and rate limiting | Clean room query engine |
| **Governance layer** | Unauthorized use, scope creep, regulatory non-compliance | Consent signal enforcement; data use agreements; query logging and audit trail; retention policies | Platform configuration and legal |

A system that implements only the data layer is a data exchange with hashing. A system that implements only the query layer is a privacy-preserving analytics tool without data security. A production clean room requires all three layers, operating in sequence.

### Differential Privacy: The Query Layer Mechanism

Differential privacy (DP) is a mathematical framework that provides a formal guarantee: the output of a query is approximately the same whether or not any individual record is included in the dataset. The guarantee is achieved by injecting calibrated random noise into query outputs before they are returned.

The noise is not random in the colloquial sense; it is drawn from a probability distribution (typically Laplace or Gaussian) calibrated to the query's sensitivity and a privacy budget parameter called epsilon (ε). Lower epsilon means more noise and stronger privacy protection; higher epsilon means less noise and better accuracy. The epsilon value is a product decision: it encodes the organization's tolerance for the tradeoff between analytical utility and privacy protection.

**What engineers need to understand about epsilon:** It is not a system parameter to be set once and forgotten. It is a privacy budget that is consumed by queries. If every query run against a dataset costs some epsilon, and the total epsilon budget is fixed, then the number of queries an analyst can run before the privacy guarantee degrades is finite. This has operational implications: clean rooms need query budgeting and monitoring, not just query execution.

### k-Anonymity: The Minimum Cohort Threshold

k-anonymity is the simpler, more widely deployed privacy control. It sets a minimum cohort size for any query result: if a query would return a count smaller than k (commonly 25 to 1,000 depending on data sensitivity), the result is suppressed rather than returned. This prevents the most common inference attack: a query that returns a count of 1 reveals exactly who is in that cohort.

k-anonymity is a floor, not a ceiling. It prevents the most egregious individual-level leakage but does not provide the formal mathematical guarantee of differential privacy. In practice, most commercial clean rooms (Snowflake Data Clean Room, Amazon Clean Rooms, LiveRamp Safe Haven) implement k-anonymity as the primary control, with differential privacy available as an additional layer for sensitive data categories.

**The healthcare and pharma implication:** Data involving health conditions, medication history, or provider visits is sensitive under both HIPAA and general privacy law. k-anonymity thresholds for healthcare data are typically set at 50 or higher; some configurations require differential privacy as a mandatory additional layer. The threshold is a product decision that must be documented and defensible to a regulator, not an engineering default.

---

## Technical Implementation

### Clean Room Architecture: The Data Flow

```python
# Clean room data preparation pipeline
# This is the data layer: hashing and tokenization before any cross-party join.
# The query layer (differential privacy, k-anonymity) is enforced by the
# clean room platform (Snowflake, AWS, LiveRamp) after ingestion.
# PEDAGOGICAL SIMPLIFICATION: production pipelines use dedicated PII vaulting
# systems (Vault by HashiCorp, AWS Secrets Manager) rather than inline hashing.

import hashlib
import hmac
import os
from dataclasses import dataclass
from typing import Optional

# The salt is an organization-specific secret.
# It prevents rainbow table attacks against hashed email addresses.
# In production: loaded from a secrets manager, never hardcoded.
HASH_SALT = os.environ.get('CLEAN_ROOM_HASH_SALT', 'replace_with_secrets_manager_value')

def hash_identifier(raw_value: str, identifier_type: str) -> str:
    """
    Produce a consistent, salted SHA-256 hash of a PII identifier.

    raw_value:       the raw email, phone, or device ID
    identifier_type: 'email', 'phone', 'device_id' — logged for audit purposes

    The same raw_value + salt always produces the same hash, enabling joins
    across parties who independently hash their own data with the same salt.
    (In practice, shared salt is established via a secure key exchange protocol;
    LiveRamp and UID2 use their own tokenization rather than shared-salt hashing.)

    Returns: lowercase hex string of the SHA-256 digest
    """
    if not raw_value or not raw_value.strip():
        raise ValueError(f"Cannot hash empty {identifier_type}")

    # Normalize before hashing: lowercase, strip whitespace
    # Normalization must be identical on both sides of the clean room join
    normalized = raw_value.strip().lower()

    hashed = hmac.new(
        key=HASH_SALT.encode('utf-8'),
        msg=normalized.encode('utf-8'),
        digestmod=hashlib.sha256
    ).hexdigest()

    return hashed


@dataclass
class CleanRoomRecord:
    """
    A single record prepared for clean room ingestion.
    Contains hashed identifiers only; no raw PII.
    """
    hashed_email: Optional[str]       # SHA-256 HMAC of normalized email
    hashed_device_id: Optional[str]   # SHA-256 HMAC of device ID
    uid2_token: Optional[str]         # UID2 / RampID token (from identity partner)
    household_id: Optional[str]       # Hashed household-level identifier
    segment_label: str                # Audience segment (no PII; behavioral label only)
    consent_signal: str               # 'opted_in', 'opted_out', 'unknown'
    data_use_permission: list         # ['targeting', 'measurement', 'attribution']
    record_date: str                  # Date of record creation (for retention enforcement)

    def is_eligible_for_clean_room(self) -> bool:
        """
        A record is eligible for clean room participation only if:
        1. It has at least one hashed identifier (email or device)
        2. The consent signal is opted_in
        3. The data use permissions include the intended use case
        Records with unknown consent are excluded by default.
        """
        has_identifier = bool(self.hashed_email or self.hashed_device_id or self.uid2_token)
        has_consent = self.consent_signal == 'opted_in'
        return has_identifier and has_consent


def prepare_publisher_segment_for_clean_room(
    raw_records: list[dict],
    intended_use: str = 'targeting'
) -> list[CleanRoomRecord]:
    """
    Transform raw publisher audience records into clean room-eligible records.

    This function is the ETL boundary: raw PII enters, hashed records exit.
    Nothing that enters this function as raw PII should leave it as raw PII.

    intended_use: the data use case being enabled ('targeting', 'measurement', 'attribution')
    """
    clean_records = []

    for record in raw_records:
        # Consent gate: exclude records without explicit opt-in
        consent = record.get('consent_signal', 'unknown')
        if consent != 'opted_in':
            continue  # Never include opted-out or unknown consent records

        # Check data use permission
        permissions = record.get('data_use_permissions', [])
        if intended_use not in permissions:
            continue  # Respect data use scope; do not expand beyond consent

        clean = CleanRoomRecord(
            hashed_email=hash_identifier(record['email'], 'email') if record.get('email') else None,
            hashed_device_id=hash_identifier(record['device_id'], 'device_id') if record.get('device_id') else None,
            uid2_token=record.get('uid2_token'),   # Already tokenized by identity partner
            household_id=hash_identifier(record['household_id'], 'household') if record.get('household_id') else None,
            segment_label=record['segment_label'], # Behavioral; not PII
            consent_signal=consent,
            data_use_permission=permissions,
            record_date=record['record_date']
        )

        if clean.is_eligible_for_clean_room():
            clean_records.append(clean)

    return clean_records
```

### SQL: Clean Room Overlap Analysis with k-Anonymity Enforcement

The query below runs inside the clean room environment after both parties have ingested their hashed records. It represents the analytical output the business needs: audience overlap by segment, with k-anonymity suppression applied before results are returned.

```sql
-- Clean room overlap analysis: publisher audience vs advertiser CRM
-- Assumes both parties have ingested hashed records into the clean room schema.
-- Written for Snowflake Data Clean Room pattern; syntax is standard SQL.
-- BigQuery Ads Data Hub: same logic; replace HAVING with QUALIFY or subquery filter.
-- Amazon Clean Rooms: same logic; k-anonymity threshold enforced via analysis rule.
-- LiveRamp Safe Haven: same logic; threshold enforced via clean room configuration.
--
-- K-ANONYMITY THRESHOLD: results with fewer than 50 matched records are suppressed.
-- Threshold is set per data collaboration agreement; healthcare data requires >= 50.
-- Standard audience data minimum is typically 25; adjust per agreement and data category.

WITH publisher_audience AS (
    -- Publisher's hashed audience records (ingested by publisher)
    SELECT
        hashed_email,
        uid2_token,
        household_id,
        segment_label,
        consent_signal,
        record_date
    FROM publisher_clean_room.audience_segments
    WHERE consent_signal = 'opted_in'
      AND 'targeting' = ANY(data_use_permissions)
      AND record_date >= DATEADD(day, -30, CURRENT_DATE)  -- Enforce retention window
),

advertiser_crm AS (
    -- Advertiser's hashed CRM records (ingested by advertiser)
    SELECT
        hashed_email,
        uid2_token,
        customer_segment,     -- e.g. 'high_value_customer', 'lapsed_buyer', 'prospect'
        purchase_category,
        crm_record_date
    FROM advertiser_clean_room.crm_segments
    WHERE consent_signal = 'opted_in'
      AND crm_record_date >= DATEADD(day, -90, CURRENT_DATE)
),

-- Match on multiple identity signals; prefer UID2 (deterministic) over hashed email
-- Hashed email is probabilistic if normalization diverges between parties.
overlap_matched AS (
    SELECT
        p.segment_label                         AS publisher_segment,
        a.customer_segment                      AS advertiser_segment,
        a.purchase_category,
        -- Record which identity signal produced the match (for quality reporting)
        CASE
            WHEN p.uid2_token    IS NOT NULL
             AND p.uid2_token    = a.uid2_token    THEN 'uid2'
            WHEN p.hashed_email  IS NOT NULL
             AND p.hashed_email  = a.hashed_email  THEN 'hashed_email'
            ELSE 'no_match'
        END                                     AS match_signal,
        p.household_id

    FROM publisher_audience p
    JOIN advertiser_crm a
      ON (
            p.uid2_token   = a.uid2_token
         OR p.hashed_email = a.hashed_email
      )
),

overlap_summary AS (
    SELECT
        publisher_segment,
        advertiser_segment,
        purchase_category,
        match_signal,
        COUNT(DISTINCT household_id)            AS matched_households,
        -- matched_records includes duplicates (one household may match via
        -- both uid2 and hashed_email); matched_households is the k-anonymity unit.
        -- The HAVING threshold below applies to matched_households (DISTINCT),
        -- not matched_records, to correctly enforce the minimum cohort size.
        COUNT(*)                                AS matched_records
    FROM overlap_matched
    WHERE match_signal != 'no_match'
    GROUP BY publisher_segment, advertiser_segment, purchase_category, match_signal
),

-- Pre-compute publisher segment totals to avoid a correlated subquery in the
-- final SELECT. A correlated subquery here would re-execute for every row in
-- overlap_summary (O(n²) behavior that times out on large datasets).
publisher_segment_totals AS (
    SELECT
        segment_label,
        COUNT(DISTINCT household_id)            AS total_households
    FROM publisher_audience
    GROUP BY segment_label
)

-- K-ANONYMITY SUPPRESSION: suppress any result below the minimum cohort threshold.
-- HAVING applied after the JOIN; standard across Snowflake, BigQuery, Redshift, Postgres.
-- Snowflake alternative: QUALIFY matched_households >= 50 (window-function context).
-- The threshold applies to matched_households (DISTINCT) not matched_records.
SELECT
    o.publisher_segment,
    o.advertiser_segment,
    o.purchase_category,
    o.match_signal,
    o.matched_households,
    -- overlap_index_pct: fraction of publisher segment that overlaps with advertiser CRM.
    -- Denominator pre-computed in publisher_segment_totals CTE; not a correlated subquery.
    ROUND(
        o.matched_households * 100.0
        / NULLIF(p.total_households, 0),
        2
    )                                           AS overlap_index_pct

FROM overlap_summary o
JOIN publisher_segment_totals p
  ON p.segment_label = o.publisher_segment

-- K-ANONYMITY: enforce minimum cohort size after join
WHERE o.matched_households >= 50  -- Adjust per data collaboration agreement

ORDER BY o.matched_households DESC;
```

### Differential Privacy: When k-Anonymity Is Not Enough

For data involving sensitive categories (health, financial, political affiliation), k-anonymity is a necessary but insufficient control. Differential privacy adds calibrated noise to query outputs, providing a formal mathematical privacy guarantee.

```python
import numpy as np
from typing import Callable

class DifferentiallyPrivateQueryEngine:
    """
    # ═══════════════════════════════════════════════════════════════════════
    # WARNING: CONCEPTUAL DEMONSTRATION ONLY. DO NOT USE IN PRODUCTION.
    # ═══════════════════════════════════════════════════════════════════════
    #
    # This class has NOT been cryptographically reviewed.
    # It does NOT implement correct sensitivity analysis.
    # It does NOT protect against floating-point side-channel attacks
    #   (an attacker can sometimes reverse-engineer the true value from
    #    the noise by exploiting IEEE 754 floating-point rounding behavior).
    # It does NOT implement correct epsilon composition across queries
    #   (the cumulative privacy cost of multiple queries requires
    #    advanced accounting beyond simple addition).
    # It does NOT handle complex query types where global sensitivity
    #   is non-trivial to compute (e.g., joins, ratios, percentiles).
    #
    # For production differential privacy, use:
    #   - OpenDP (opendp.org): academic-reviewed, open-source, peer-audited
    #   - Google's DP Library (github.com/google/differential-privacy)
    #   - Apple's TensorFlow Privacy or CMSNoise
    #
    # This code exists solely to illustrate the epsilon budget concept
    # to product managers and engineers who need to understand WHY
    # differential privacy matters as a product requirement — not HOW
    # to implement it correctly.
    # ═══════════════════════════════════════════════════════════════════════

    A note on positioning: I am not a cryptographer. I cannot vouch for the
    mathematical correctness of a DP implementation. What a product manager
    can and should do is:
      1. Understand why epsilon budget tracking is a product requirement
         (queries consume budget; budgets expire; this has operational consequences)
      2. Know which questions to ask a cryptographer before approving a DP system
         (see "What PMs Should Ask Cryptographers" section below)
      3. Validate that a production DP implementation meets the business need
         (correct sensitivity analysis for the query types in use; proper
          composition accounting; audit trail for epsilon consumption)

    The code below demonstrates the concept only. Do not ship it.
    """

    def __init__(self, epsilon_budget: float = 1.0):
        """
        epsilon_budget: the total privacy budget for this clean room session.
        Lower epsilon = more noise = stronger privacy = less analytical utility.
        
        Typical ranges:
          epsilon < 0.1:  very strong privacy; results may be noisy for small cohorts
          epsilon = 1.0:  standard for most commercial clean room use cases
          epsilon > 10.0: weak privacy; reserved for highly aggregated, low-sensitivity data
        
        The budget is consumed by each query. When budget is exhausted, no further
        queries are permitted until the budget is reset (typically at session end).
        Budget tracking is a product requirement, not an engineering default.
        """
        self.epsilon_budget = epsilon_budget
        self.epsilon_consumed = 0.0

    def _laplace_noise(self, sensitivity: float, epsilon: float) -> float:
        """
        Sample from Laplace distribution with scale = sensitivity / epsilon.
        sensitivity: the maximum amount a single record can change the query result.
                     For COUNT queries, sensitivity = 1.
                     For SUM queries, sensitivity = max possible value per record.
        """
        scale = sensitivity / epsilon
        return np.random.laplace(loc=0, scale=scale)

    def private_count(
        self,
        true_count: int,
        epsilon_cost: float = 0.1,
        sensitivity: float = 1.0
    ) -> dict:
        """
        Return a differentially private count with Laplace noise.
        Returns the noisy result along with budget tracking metadata.
        """
        if self.epsilon_consumed + epsilon_cost > self.epsilon_budget:
            raise PrivacyBudgetExhaustedError(
                f"Query would consume {epsilon_cost} epsilon; "
                f"only {self.epsilon_budget - self.epsilon_consumed:.3f} remaining. "
                f"Session must be reset before additional queries are permitted."
            )

        noise = self._laplace_noise(sensitivity, epsilon_cost)
        noisy_count = max(0, round(true_count + noise))  # Counts cannot be negative

        self.epsilon_consumed += epsilon_cost

        return {
            'result': noisy_count,
            'epsilon_cost': epsilon_cost,
            'epsilon_remaining': round(self.epsilon_budget - self.epsilon_consumed, 4),
            'privacy_mechanism': 'laplace',
            'note': 'Result includes calibrated noise per differential privacy guarantee. '
                    'Individual counts are not reliable; use for segment-level analysis only.'
        }


class PrivacyBudgetExhaustedError(Exception):
    """Raised when a query would exceed the session's epsilon budget."""
    pass
```

### What Product Managers Should Ask Cryptographers

Before approving a differential privacy implementation for production, a product manager should be able to ask these questions and evaluate the answers; not implement the answers themselves:

| Question | Why It Matters |
|----------|---------------|
| "What is the global sensitivity for each query type we run?" | Sensitivity determines noise scale; wrong sensitivity = wrong privacy guarantee |
| "How are you handling epsilon composition across a session?" | Simple addition underestimates cumulative privacy cost; advanced composition theorems exist |
| "What is your approach to floating-point side-channel protection?" | IEEE 754 rounding can leak information; production libraries account for this |
| "Have you run this implementation against the OpenDP validator?" | OpenDP provides formal proof verification; unvalidated implementations should not ship |
| "What is our epsilon budget policy, and who approves budget resets?" | Budget reset is a business decision with privacy implications; it needs a named owner |
| "How does the system behave when the budget is exhausted: fail closed or degrade?" | Fail closed (no queries) is safer; degraded mode may leak information if not designed carefully |
| "What data categories require lower epsilon (more noise) in our deployment?" | Health, financial, and political data typically require tighter budgets than behavioral data |

The goal of these questions is not to test the cryptographer; it is to ensure that the product manager understands the constraints well enough to scope the feature correctly, communicate the operational implications to stakeholders, and recognize when the implementation diverges from the privacy guarantee the business has committed to clients.

**What the log schema must carry for clean room governance:**

```
query_id               (string):   unique identifier for each clean room query
session_id             (string):   groups related queries for budget tracking
submitting_party       (string):   'publisher' or 'advertiser'; audit trail
query_timestamp        (datetime): UTC timestamp of query submission
query_text_hash        (string):   SHA-256 hash of the query (not raw SQL; protects confidentiality)
result_row_count       (int):      number of rows returned (after suppression)
rows_suppressed        (int):      number of rows withheld due to k-anonymity threshold
epsilon_cost           (float):    epsilon consumed by this query (if DP enabled)
epsilon_remaining      (float):    epsilon budget remaining after this query
k_threshold_applied    (int):      the k-anonymity threshold in effect
data_use_case          (string):   'targeting', 'measurement', 'attribution'
consent_scope_verified (bool):     whether consent signals were validated before query execution
```

---

## Failure Mode

**What breaks when engineers treat "encrypted" as "private":**

The most common clean room failure mode in adtech environments is a system that hashes PII at the point of data transfer and returns exact query results without any k-anonymity or differential privacy enforcement. This system is compliant in a narrow technical sense: raw email addresses do not cross organizational boundaries. It is not compliant in the broader regulatory and commercial sense, because the query outputs leak information about individuals through the differencing attacks described above.

The commercial consequence is deferred but severe. Advertisers and publishers who enter data collaboration agreements typically include representations about privacy controls: each party represents that their clean room implementation meets industry standards. When a regulatory audit reveals that the clean room returns exact counts on cohorts of three or four records, both parties face exposure. The partnership unwinds. The premium CPM that justified the collaboration disappears.

The second failure mode is scope creep without governance. A clean room established for targeting (matching the advertiser's CRM against the publisher's audience segments) may gradually be used for competitive intelligence (submitting queries that reveal the publisher's total first-party data volume, segment composition, or content consumption patterns). Without query auditing and data use enforcement, scope creep is invisible until it becomes a dispute.

A third failure mode is specific to healthcare and pharmaceutical environments: a clean room that does not enforce consent scope at the record level. A patient who consented to their data being used for "personalized health content recommendations" has not consented to that data being joined against a pharmaceutical advertiser's prescription CRM for targeting purposes. The consent scope must be enforced per record in the clean room ETL pipeline; not assumed at the dataset level.

In DOOH environments, the consent problem is structurally different and in some ways harder. A patient in a medical office waiting room has not opted in to ad targeting; they are simply present. The "consent" implied by physical presence in a venue is not equivalent to the explicit opt-in required for data use under GDPR Article 6 or CPRA. This places additional burden on the governance layer: location-based audience building that uses venue presence as a proxy for consent may implicate protected health information under HIPAA if the venue type (medical office, infusion center, pharmacy) itself constitutes a sensitive data category. The data use agreement and the ETL consent gate must account for this; it is not a question an engineer can resolve by looking at the data schema alone.

This pattern has been observed across multiple adtech and healthcare data collaboration deployments. The engineering response to "just hash the data" is understandable given the complexity of differential privacy and k-anonymity; but the commercial and regulatory consequence of that simplification grows as data collaboration becomes more central to publisher revenue strategy.

---

## Translation Conversation: Product and Engineering

*How to explain the difference between encryption and privacy-preserving computation:*

"Imagine you want to know how many people in a city have a specific medical condition, but you're not allowed to look at anyone's medical records. One approach: encrypt all the records and look at the encrypted versions. But encrypting a record doesn't tell you anything; you can't count what you can't read. The second approach: have a trusted third party count the records and tell you only the total. That's k-anonymity: you only get the answer if enough records exist to make the count meaningless for identifying individuals.

The problem is that if you can ask many questions, you can eventually narrow things down: 'How many people in zip code X have condition Y?' then 'How many people in zip code X who are female have condition Y?' If the second answer is one less than the first, you know exactly who the one man in that cohort is. That's the differencing attack that differential privacy prevents: it adds noise to every answer so that no sequence of queries can narrow in on an individual.

For our clean room: hashing the email addresses protects the raw data. But if we return exact query counts, we're vulnerable to differencing attacks on small cohorts. The k-anonymity threshold is the minimum we need to suppress the most obvious attacks. Differential privacy is what we need when the data involves health or financial information where even probabilistic inference is a problem."

---

## Open-Source Implementation

The Signal CTV PRD (`github.com/peterduhon/intentsegments`: Signal CTV v2 module) documents the privacy compliance architecture that this artifact explains at the conceptual and technical level. Specifically:

**Section 6 of the Signal CTV PRD** (Privacy and Compliance Architecture) specifies: no third-party identifiers in the pipeline; SHA-256 hashed email only in matching; clean room partner matching with no raw PII shared; consent signal enforcement from the publisher CMP; 30-day rolling retention window. These are the data layer and governance layer controls described in the Three Layers table above.

**The enrichment pipeline simulation** in Signal CTV models the sequential match rate lift from layered first-party data sources, including a clean room partner layer that contributes an incremental 8 percentage points of household match rate. The clean room layer in that simulation is the practical instantiation of the query layer architecture described in this artifact.

**Media Publisher Identity Graph Explorer** (`github.com/peterduhon/media-publisher-identity-graph-explorer`) shows the Snowflake Clean Room as an active integration partner with an 89% match rate and real-time latency; the highest match rate in the integration stack as modeled in the repo's synthetic partner data. (Note: this figure is drawn from the simulated partner data in the Identity Graph Explorer repo, not from a live production deployment. It is representative of published Snowflake Data Clean Room match rate benchmarks for authenticated household data; verify against current Snowflake documentation before citing in a client context.) This reflects the commercial reality that clean room-mediated identity resolution outperforms probabilistic graph inference for the cohorts where both parties have authenticated data.

The next milestone for the intentsegments repo extends the privacy architecture documentation to include the k-anonymity threshold logic and the epsilon budget tracking described in this artifact, making the PRD and the code mutually reinforcing.

---

## Chapter 1: AdTech Translation — Full Series

| # | Concept | Technical Focus | Status |
|---|---------|-----------------|--------|
| 01 | [First-Price vs. Second-Price Auction](./01-first-price-second-price.md) | Bid shading; floor price optimization; log schema price fields | ✅ Published |
| 02 | [Viewability (MRC / IAS / MOAT)](./02-viewability.md) | Pixel firing; viewport detection; SSAI completion beacons; DOOH OTS | ✅ Published |
| 03 | [Frequency Capping](./03-frequency-capping.md) | Household graph deduplication; cross-device session management; DOOH screen-level vs audience-level | ✅ Published |
| 04 | [Attribution Window](./04-attribution-window.md) | Conversion pixel timing; post-click vs. post-view; consideration cycle configuration | ✅ Published |
| **05** | **Clean Rooms and PETs** | Differential privacy; k-anonymity; query logging; enabling queries without exposing raw data | ✅ Published |
| 06 | SLA and Uptime Guarantees | Error budgets; circuit breakers; graceful degradation; contract-level implications | 📋 Planned |

Chapters 2 through 4 extend The Consensus Layer beyond adtech translation into design intelligence, API-first product methodology, and enterprise product infrastructure. See the [repository README](../README.md) for the full project map.

---

## References and Further Reading

- IAB Tech Lab: "Data Collaboration Standards" — clean room interoperability requirements and data use agreement framework
- MRC: "Privacy-Safe Measurement Standards" — k-anonymity and differential privacy guidance for audience measurement
- OpenDP (opendp.org): open-source differential privacy library — the reference implementation for production DP in data science contexts; do not build custom DP without consulting this first
- Google: "Differential Privacy Library" (github.com/google/differential-privacy) — production-grade DP for SQL and data pipelines
- Snowflake: "Data Clean Room" documentation — commercial implementation of k-anonymity thresholds and query governance in Snowflake Horizon
- Amazon: "AWS Clean Rooms" documentation — k-anonymity analysis rules and query logging architecture
- LiveRamp: "Safe Haven" documentation — pseudonymization, UID2 tokenization, and clean room federation across partners
- IAPP: "Privacy-Enhancing Technologies" overview — regulatory context for PETs under GDPR Article 25 (data protection by design) and CPRA
- The Consensus Layer Artifact 03: Frequency Capping — clean room identity resolution and household graph data share the same privacy architecture; consent signals enforced in the clean room ETL must also gate household graph lookups
- The Consensus Layer Artifact 04: Attribution Window — view-through attribution in healthcare environments must run inside a privacy-preserving computation layer; the schema fields `holdout_group` and `venue_type` in Artifact 04 may constitute sensitive data categories requiring DP enforcement in Artifact 05's architecture

---

*The Consensus Layer is a living reference built by Pete Duhon. Born in AdTech. Built for product teams translating business logic into technical infrastructure.*
*GitHub: github.com/peterduhon | Last updated: June 2026*
