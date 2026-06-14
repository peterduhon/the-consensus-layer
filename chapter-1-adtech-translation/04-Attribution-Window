# 04: Attribution Window

**The Consensus Layer: Chapter 1 — AdTech Translation**
*A shared language for product teams translating business logic into technical infrastructure.*

---

## The Business Problem

Every advertiser running a campaign needs to answer one question: did the ad work? Attribution is the system that attempts to answer it by connecting an ad exposure to a downstream outcome: a purchase, a registration, a prescription request, a store visit. The attribution window is the time period within which an exposure is eligible to receive credit for an outcome that follows it.

Get the window right and advertisers can measure the actual causal relationship between their spend and their results. Get it wrong in either direction and the measurement system either overclaims (crediting ads for outcomes that would have happened anyway) or underclaims (missing the influence of ads that worked slowly). In both cases, the advertiser makes the wrong budget decision in the next planning cycle.

Attribution window design is not a technical preference; it is a product decision with direct revenue implications. The wrong window produces wrong measurement, which produces wrong spend allocation, which produces wrong renewal conversations.

---

## What Engineers Think

Engineers with data pipeline backgrounds often begin by modeling attribution as a temporal join: match conversion events to impression events within a time window on a shared identifier. This is the correct starting point. The join pattern is sound; the tooling to implement it is well-understood; and the concept of a lookback window is intuitive.

The problems emerge in what the join assumes: that the identifier is stable across channels, that the window length is correct for all product categories, and that credit is a scalar value rather than a model-dependent allocation. These are not engineering errors; they are product decisions that have been implicitly embedded in the technical specification without being named as such.

**What the join assumes:** that there is a persistent user identifier linking the impression to the conversion. On web, this is a cookie or a hashed email. On mobile, it is a device ID. On CTV, as established in Artifact 03, there is no cookie; the identifier is a device ID or a household graph resolution. In healthcare and pharmaceutical DOOH environments, there may be no digital identifier at all connecting the viewer of an out-of-home ad to their subsequent prescription request.

**What the join leaves unspecified:** the business logic that determines which window length is appropriate for which product category; which attribution model (last-touch, first-touch, linear, time-decay) reflects the actual purchase decision process; and how to handle multi-touch paths where multiple channels each claim credit for the same conversion. These are product questions that determine what the join parameters should be. When they go unspecified, engineers apply defaults; and defaults are rarely calibrated to any particular advertiser's consideration cycle.

A technically correct pipeline with the wrong window and the wrong model produces commercially meaningless outputs: a 7-day last-touch attribution applied to a pharmaceutical product with a 90-day consideration cycle will show near-zero attributed conversions, leading the client to conclude the campaign failed when the measurement was simply wrong for the product.

---

## What the Business Actually Needs

### Attribution Models: Which Credit Rule Applies

Before the window question, the model question: how is credit distributed across the touchpoints that precede a conversion?

| Model | Credit Rule | When It Fits | When It Misleads |
|-------|-------------|-------------|------------------|
| **Last-touch** | 100% credit to the final touchpoint before conversion | Direct response campaigns; short consideration cycles; e-commerce | Brand campaigns; long consideration cycles; upper-funnel media |
| **First-touch** | 100% credit to the first touchpoint | Awareness measurement; understanding what introduces a customer to a brand | Ignores the role of lower-funnel media in closing the conversion |
| **Linear** | Equal credit across all touchpoints in the window | Multi-channel campaigns where all touches are considered equal | Overvalues low-impact touches; undervalues high-impact ones |
| **Time-decay** | More credit to touchpoints closer to conversion | Products with a clear final decision moment | Systematically undervalues CTV and upper-funnel brand media |
| **Data-driven** | ML-derived credit weights based on actual path analysis | Large campaigns with sufficient conversion volume for model training | Requires minimum conversion thresholds; opaque to clients without model transparency |

**The CTV implication:** CTV advertising is almost entirely an upper-funnel and mid-funnel medium. There is no click on a television. A viewer who sees an ad on their Roku and then purchases the product three weeks later will not appear in a last-touch attribution model because the last touch was likely a search ad or a direct website visit. Last-touch models systematically undervalue CTV. This is not a measurement edge case; it is the reason the industry developed view-through attribution as a first-class model for video and CTV environments.

### Attribution Window Length: The Consideration Cycle Determines the Window

The correct window length is not a technical constant; it is derived from how long the target audience typically takes to move from awareness to action for a given product category.

| Product Category | Typical Consideration Cycle | Appropriate Attribution Window | Why |
|-----------------|---------------------------|-------------------------------|-----|
| **CPG / Impulse purchase** | Hours to days | 1 to 7 days post-click; 1 day post-view | Low price point; immediate purchase decision |
| **Automotive** | Weeks to months | 30 to 90 days | High price point; research-intensive; multiple stakeholders |
| **Direct-to-consumer pharma (DTC)** | Days to weeks | 14 to 30 days | Patient must see a physician; prescription adds a step |
| **Healthcare professional (HCP) pharma** | Weeks to months | 30 to 90 days | Physician awareness to prescription behavior change is slow |
| **Financial services** | Weeks to months | 30 to 60 days | High trust threshold; comparison shopping; account opening lag |
| **B2B SaaS** | Months | 60 to 90 days | Multi-stakeholder; procurement cycles; contract review |
| **Healthcare DOOH (point-of-care)** | Variable | 7 to 30 days | Patient sees ad in waiting room; may act on next prescription visit |

**The critical mistake in healthcare and pharmaceutical environments:** Engineers who configure a standard 7-day post-impression window for a pharmaceutical campaign will systematically miss attributed conversions. A patient who sees an HCP-directed ad in a medical office waiting room and then asks their doctor about the medication at a follow-up appointment two weeks later falls outside a 7-day window entirely. The campaign reports near-zero conversions. The advertiser reallocates budget away from the channel. The measurement failure masquerades as a campaign performance failure.

**Point-of-care DOOH adds another layer:** In healthcare DOOH environments, the screen is in the patient's path at a specific moment (waiting room, pharmacy, infusion center). The temporal relationship between exposure and outcome is not just about window length; it is about whether the measurement system can connect a physical ad exposure to a downstream digital or prescription signal at all. This requires either: a mobile device pairing signal (the patient's phone was at the same location as the screen at the time of the play); a pharmacy data integration (script lift measurement against a control geography); or a patient survey panel. None of these are "just a join."

> **Research note:** The pharmaceutical and healthcare attribution patterns described in this document are derived from industry research, regulatory guidance (FDA, HIPAA), and interviews with healthcare adtech practitioners. They represent the current state of the field as understood by the author. Direct production implementation experience in pharmaceutical attribution varies; readers should validate specific window configurations and compliance requirements against their own regulatory context and measurement vendor guidance.

### Post-Click vs. Post-View: The Two Attribution Modes

**Post-click attribution:** Credit flows from a user's explicit click on an ad to a subsequent conversion within the window. On web and mobile, this is the default: the click is a deterministic signal of intent. The user chose to engage; the attribution is defensible.

**Post-view (view-through) attribution:** Credit flows from an ad impression (without a click) to a subsequent conversion within the window, if the impression happened before the conversion within the attribution window. This is the model for CTV and video environments where clicking is not possible, and for brand advertising on any channel where the goal is awareness and consideration rather than immediate response.

View-through attribution is more contested than post-click attribution because it cannot distinguish between ads that influenced the conversion and ads that simply ran near a conversion that would have happened regardless. The industry mitigation is holdout testing: a matched control group that does not receive the ad. If the exposed group converts at a higher rate than the holdout, the delta is the attributable lift. Without a holdout, view-through attribution is an assumption, not a measurement.

---

## Technical Implementation

### The Attribution Log Schema: What Must Be Captured at Impression Time

Attribution is a retrospective join: the conversion event that arrives later is matched against impression events that were logged earlier. If the impression log does not carry the fields needed to make that join possible, the attribution system cannot be built retroactively. The schema decision at impression time determines what can be measured downstream.

```python
# Attribution-ready impression log: required fields
# These fields must be written at impression time, not derived later.
# A schema that omits any of these cannot support post-campaign attribution joins.

from dataclasses import dataclass
from typing import Optional
from datetime import datetime

@dataclass
class AttributionReadyImpression:
    # Core identifiers
    impression_id: str              # Unique impression ID
    campaign_id: str
    creative_id: str
    placement_id: str

    # Identity fields for attribution join
    household_id: Optional[str]     # Resolved household ID (CTV/cross-device)
    device_id: str                  # Device-level ID (RIDA, Fire TV ID, cookie)
    user_id_hash: Optional[str]     # Hashed email or UID2 if authenticated
    ip_address_hash: str            # Hashed IP for probabilistic matching

    # Temporal fields: both are required for window calculation
    impression_timestamp: datetime  # Exact UTC timestamp of impression delivery
    event_date: str                 # Partition date (YYYY-MM-DD) for query efficiency

    # Channel and environment fields: required for model-specific window assignment
    channel: str                    # 'CTV', 'display', 'video', 'DOOH', 'mobile'
    deal_type: str                  # 'open_auction', 'PMP', 'PG', 'direct'
    inventory_type: str             # 'pre_roll', 'mid_roll', 'banner', 'OOH'

    # Attribution model fields
    attribution_eligible: bool      # Whether this impression is in the attribution pool
    attribution_window_days: int    # Configured window for this campaign/channel
    attribution_model: str          # 'last_touch', 'linear', 'time_decay', 'view_through'
    viewable: Optional[bool]        # For view-through: was the ad viewable/completed?
    completion_rate: Optional[float] # CTV: Q4 beacon fired = 1.0; partial = 0.25/0.5/0.75

    # Healthcare/DOOH-specific fields
    location_id: Optional[str]      # DOOH: physical screen location identifier
    venue_type: Optional[str]       # DOOH: 'medical_office', 'pharmacy', 'hospital_waiting'
    mobile_device_paired: Optional[bool]  # DOOH: was a mobile device co-located at serve time?

    # Holdout and test fields
    holdout_group: Optional[bool]   # True = control group; False = exposed group
    experiment_id: Optional[str]    # A/B or holdout experiment ID

    # Note: all PII fields (IP, user ID) must be hashed at write time.
    # Raw IP addresses and email addresses must never enter the impression log.
    # Healthcare environments require additional PHI compliance review;
    # venue_type and location_id may constitute sensitive data under HIPAA in some configurations.
```

### Attribution Join: The Core Query Pattern

```sql
-- View-through attribution join: CTV impressions to downstream conversions
-- Written for Snowflake syntax
-- BigQuery: replace DATEADD with DATE_SUB / DATE_ADD; INTERVAL syntax differs
-- Redshift: DATEADD compatible; adjust lateral join syntax if used

-- ─────────────────────────────────────────────────────────────────────────────
-- QUERY A: Attribution Join
-- Match CTV impressions to downstream conversions within the configured window.
-- Written for Snowflake syntax.
-- BigQuery: replace DATEADD with DATE_SUB/DATE_ADD; TIMESTAMP_DIFF for DATEDIFF
-- Redshift: DATEADD compatible; ROW_NUMBER window function supported
-- Postgres: replace DATEADD with CURRENT_DATE - INTERVAL; ROW_NUMBER supported
--
-- NOTE on identity join: The three-way OR join below
-- (household_id OR user_id_hash OR ip_address_hash) can produce duplicate
-- attribution paths if a conversion matches on more than one identity field.
-- Production systems typically attempt deterministic match first, then
-- probabilistic, with match quality logged per attempt and a priority hierarchy
-- enforced before the deduplication step. This query uses ROW_NUMBER to select
-- the highest-quality match per conversion, which handles the duplicate case.
-- ─────────────────────────────────────────────────────────────────────────────

WITH conversion_events AS (
    SELECT
        conversion_id,
        household_id,
        user_id_hash,
        ip_address_hash,
        conversion_timestamp,
        conversion_type,
        conversion_value,
        campaign_id
    FROM conversion_log
    WHERE event_date >= DATEADD(day, -90, CURRENT_DATE)
      AND campaign_id = :campaign_id
),

impression_candidates AS (
    SELECT
        i.impression_id,
        i.campaign_id,
        i.household_id,
        i.user_id_hash,
        i.ip_address_hash,
        i.impression_timestamp,
        i.channel,
        i.completion_rate,
        i.attribution_window_days,
        i.attribution_model,
        i.holdout_group,
        i.venue_type
    FROM attribution_ready_impressions i
    WHERE i.event_date >= DATEADD(day, -90, CURRENT_DATE)
      AND i.campaign_id = :campaign_id
      AND i.attribution_eligible = TRUE
      AND i.completion_rate >= 0.75
),

attributed_raw AS (
    -- Three-way OR join; may produce duplicates if a conversion matches
    -- on more than one identity field. Deduplicated below via ROW_NUMBER.
    SELECT
        c.conversion_id,
        c.conversion_timestamp,
        c.conversion_type,
        c.conversion_value,
        i.impression_id,
        i.impression_timestamp,
        i.channel,
        i.completion_rate,
        i.attribution_model,
        i.holdout_group,
        i.venue_type,
        DATEDIFF('hour', i.impression_timestamp, c.conversion_timestamp) AS hours_to_conversion,
        CASE
            WHEN i.household_id    = c.household_id    THEN 1
            WHEN i.user_id_hash    = c.user_id_hash    THEN 2
            WHEN i.ip_address_hash = c.ip_address_hash THEN 3
        END AS identity_match_quality
    FROM conversion_events c
    JOIN impression_candidates i
      ON (
            i.household_id    = c.household_id
         OR i.user_id_hash    = c.user_id_hash
         OR i.ip_address_hash = c.ip_address_hash
      )
      AND i.impression_timestamp < c.conversion_timestamp
      AND i.impression_timestamp >= DATEADD(
            'day', -i.attribution_window_days, c.conversion_timestamp
          )
),

-- Last-touch deduplication using ROW_NUMBER (portable across Snowflake, BigQuery, Redshift)
-- To switch to first-touch: change ORDER BY impression_timestamp DESC to ASC
-- To switch to linear: skip deduplication and weight each row by 1/COUNT(*) per conversion
last_touch AS (
    SELECT *
    FROM (
        SELECT
            *,
            ROW_NUMBER() OVER (
                PARTITION BY conversion_id
                ORDER BY
                    identity_match_quality ASC,   -- Prefer higher-quality identity match
                    impression_timestamp DESC      -- Then most recent impression
            ) AS rn
        FROM attributed_raw
    ) ranked
    WHERE rn = 1
)

SELECT
    channel,
    venue_type,
    attribution_model,
    identity_match_quality,
    COUNT(DISTINCT conversion_id)       AS attributed_conversions,
    ROUND(SUM(conversion_value), 2)     AS attributed_revenue,
    ROUND(AVG(hours_to_conversion), 1)  AS avg_hours_to_conversion
FROM last_touch
GROUP BY channel, venue_type, attribution_model, identity_match_quality
ORDER BY attributed_conversions DESC;


-- ─────────────────────────────────────────────────────────────────────────────
-- QUERY B: Holdout Lift Analysis
-- Run separately from the attribution join for computational efficiency.
-- Requires holdout_group field in the impression log (TRUE = control, FALSE = exposed).
-- Also requires a conversion_flag field in the impression log (1 if conversion occurred,
-- 0 if not) so we can compute conversion rates across exposed and control populations.
--
-- Production note: holdout analysis typically runs against the full campaign population
-- (all impressions, exposed and control), not just the attributed subset.
-- Run this query against the full impression log, not the last_touch CTE above.
-- ─────────────────────────────────────────────────────────────────────────────

SELECT
    channel,
    holdout_group,
    COUNT(DISTINCT household_id)                              AS households,
    SUM(conversion_flag)                                      AS conversions,
    ROUND(
        SUM(conversion_flag) * 1.0
        / NULLIF(COUNT(DISTINCT household_id), 0),
        6
    )                                                         AS conversion_rate,

    -- Lift is computed externally by comparing exposed_rate - control_rate.
    -- Keeping exposed and control as separate rows (rather than a single lift column)
    -- makes the underlying rates visible and auditable, which matters for client reporting.
    ROUND(AVG(win_price), 4)                                  AS avg_cpm

FROM attribution_ready_impressions
WHERE event_date >= DATEADD(day, -90, CURRENT_DATE)
  AND campaign_id = :campaign_id
  AND attribution_eligible = TRUE

GROUP BY channel, holdout_group
ORDER BY channel, holdout_group;
```

**Why these are two queries, not one:** Attribution joins and holdout lift analyses operate on different populations (attributed impressions vs. all impressions) and at different computational scales. Combining them in a single query obscures both the attribution logic and the lift calculation, and produces the specific error where a count is divided by itself. Keeping them separate makes each query auditable, testable, and independently explainable to a client.

**What Query A produces:** Attributed conversions by channel, identity match quality, and venue type, with average time-to-conversion. The `identity_match_quality` column makes the attribution auditable: a household-level match (quality 1) carries more evidentiary weight than a probabilistic IP match (quality 3).

**What Query B produces:** Conversion rates for exposed and control populations, separated by row rather than collapsed into a single lift column. The lift is computed externally (exposed rate minus control rate) rather than inside SQL, which makes the underlying rates visible to the client rather than presenting only the derived delta.

### Consideration Cycle Calibration: Setting the Right Window

```python
# Attribution window configuration by product category and channel
# This logic belongs in the campaign configuration layer, not hardcoded in the SQL.
# Product managers define these windows; engineers implement the config system.

# ATTRIBUTION_WINDOWS: default configuration table
#
# OWNERSHIP: Product management owns the values in this table.
# Engineering owns the configuration system that stores and serves them.
# These are not the same responsibility.
#
# In production, this table is stored in a configuration database with:
#   - Per-campaign override capability (individual advertisers differ within a category)
#   - A/B test cell assignment for window length experiments
#   - Client-specific configuration with approval workflow
#   - Audit trail for all changes (regulatory requirement in healthcare and pharma)
#   - Version history with rollback support
#
# The values below are starting points derived from industry research and
# should be validated against actual conversion path data per client.
# They should be reviewed by product management quarterly and adjusted
# when measurement studies reveal systematic over- or under-attribution.
#
# Product decisions embedded as code constants are invisible to the business.
# This table should be visible, versioned, and owned by a named product stakeholder.
ATTRIBUTION_WINDOWS = {
    # (product_category, channel): (post_click_days, post_view_days)
    ('cpg', 'display'):          (7,   1),
    ('cpg', 'CTV'):              (7,   1),
    ('automotive', 'CTV'):       (90, 30),
    ('automotive', 'display'):   (30,  7),
    ('pharma_dtc', 'CTV'):       (30, 14),   # DTC: patient must see MD; add step
    ('pharma_dtc', 'DOOH'):      (30, 14),   # Waiting room exposure; next appointment
    ('pharma_hcp', 'DOOH'):      (90, 30),   # HCP consideration cycle is longer
    ('pharma_hcp', 'display'):   (60, 21),
    ('financial_services', 'CTV'):   (60, 21),
    ('financial_services', 'display'): (30, 7),
    ('healthcare_general', 'DOOH'):   (30, 14),
}

def get_attribution_window(
    product_category: str,
    channel: str,
    attribution_type: str  # 'post_click' or 'post_view'
) -> int:
    """
    Return the appropriate attribution window in days for a given
    product category, channel, and attribution type.

    This function encodes product and business logic, not engineering logic.
    The values should be reviewed by product management on a quarterly basis
    and adjusted based on observed conversion path data.

    For channels without click capability (CTV, DOOH), post_click_days
    should return 0 or raise a configuration error; post_view is the
    only applicable model.
    """
    key = (product_category.lower(), channel.upper())
    windows = ATTRIBUTION_WINDOWS.get(key)

    if windows is None:
        # Default fallback: standard industry windows
        # Log a warning: unconfigured category/channel combination
        # should be reviewed by product management
        return 7 if attribution_type == 'post_click' else 1

    post_click_days, post_view_days = windows
    return post_click_days if attribution_type == 'post_click' else post_view_days
```

---

## Failure Mode

**What breaks when engineers apply the wrong attribution window:**

The most common failure mode is category-agnostic window configuration: the engineering team sets a single 7-day post-impression window for all campaigns, all channels, and all product categories, because 7 days is the most common industry default and no one specified otherwise. For CPG and direct response campaigns, this window is approximately correct. For everything else on the spectrum, it is wrong in ways that produce specific, predictable errors.

For pharmaceutical campaigns targeting healthcare professionals through DOOH environments: the 7-day window misses the majority of attributable conversions. An HCP who sees a branded medication ad in a hospital waiting room and then begins prescribing that medication three weeks later is not captured. The campaign's measured return on ad spend is near zero. The advertiser concludes that the channel does not work and reduces investment. The platform loses a category of advertiser spend that is historically among the highest-CPM inventory in digital advertising.

For CTV upper-funnel brand campaigns: the 7-day window captures the short consideration buyers (who would have been captured by search regardless) and misses the long-consideration buyers who saw the CTV ad during the awareness phase. A 30-day post-view window with holdout measurement would show statistically significant lift. The 7-day window shows noise. The same misattribution problem drives the persistent undervaluation of CTV relative to performance channels in mixed-media models.

For cross-channel campaigns: when a single 7-day window is applied uniformly and last-touch credit goes to the final search or display touchpoint, CTV and DOOH systematically receive zero attributed credit even when they were the first-touch channel that initiated the consideration journey. Mixed-media models built on this data underinvest in upper-funnel media and overinvest in lower-funnel media; the budget eventually concentrates in channels that close conversions rather than channels that generate them.

This pattern has been observed across multiple CTV and healthcare DOOH environments. The window configuration is typically set once during platform launch and reviewed infrequently because the engineering team treats it as a system parameter rather than a product decision. Product managers who inherit these systems often discover the misconfiguration only when a client runs an independent measurement study that produces results inconsistent with the platform's reported attribution.

---

## Translation Conversation: Product and Engineering

*How to explain attribution window misconfiguration to a platform engineer in concrete terms:*

"Imagine you're measuring whether a billboard on a highway causes people to visit a car dealership. You decide to count every dealership visit that happens within 24 hours of someone driving past the billboard. You measure for three months and find almost zero attributed visits. You conclude billboards don't work. But what if most people who see a car ad on a highway visit the dealership in the next two to four weeks, not the next day? The billboard worked; your measurement window was wrong.

That is the attribution window problem in advertising. For a pharmaceutical brand trying to reach patients who then have to schedule a doctor's appointment, get a prescription, and fill it at a pharmacy, a 7-day window misses the entire conversion pathway. The measurement says the ad failed. The brand pulls budget. The platform loses the category.

The window is a product decision, not a system constant. It should be set per campaign, per product category, and per channel. Our configuration system needs to support that: not one global window, but a table of windows that can be tuned based on what we know about how long it takes each type of advertiser's customer to make a decision."

---

## Open-Source Implementation

**AdInsight: Reddit Ad Dashboard** (`github.com/peterduhon/adinsight`) implements a Streamlit-based campaign performance interface backed by SQLite. The current schema includes impression and engagement events but does not carry the attribution-ready fields documented in this artifact (specifically: `attribution_window_days`, `attribution_model`, `holdout_group`, `completion_rate` at the impression level). The next milestone for AdInsight extends the SQLite schema to carry these fields and adds a view-through attribution report tab that implements the SQL join pattern above against the simulated impression and conversion logs.

**AdVantageX RTB Simulation** (`github.com/peterduhon/rtb-practice`) generates synthetic bid request and impression data for OpenRTB auction simulation. The attribution schema extension in this artifact connects naturally to that simulation: synthetic conversion events (with configurable delay distributions that mirror the consideration cycles above) can be generated against the existing impression log to produce a realistic attribution dataset for demonstration purposes.

---

## Chapter 1: AdTech Translation — Full Series

| # | Concept | Technical Focus | Status |
|---|---------|-----------------|--------|
| 01 | [First-Price vs. Second-Price Auction](./01-first-price-second-price.md) | Bid shading; floor price optimization; log schema price fields | ✅ Published |
| 02 | [Viewability (MRC / IAS / MOAT)](./02-viewability.md) | Pixel firing; viewport detection; SSAI completion beacons; DOOH OTS | ✅ Published |
| 03 | [Frequency Capping](./03-frequency-capping.md) | Household graph deduplication; cross-device session management; DOOH screen-level vs audience-level | ✅ Published |
| **04** | **Attribution Window** | Conversion pixel timing; post-click vs. post-view logic; consideration cycle configuration | ✅ Published |
| 05 | Clean Rooms and PETs | Differential privacy; k-anonymity; query logging; privacy-preserving computation | 📋 Planned |
| 06 | SLA and Uptime Guarantees | Error budgets; circuit breakers; graceful degradation; contract-level implications | 📋 Planned |

Chapters 2 through 4 extend The Consensus Layer beyond adtech translation into design intelligence, API-first product methodology, and enterprise product infrastructure. See the [repository README](../README.md) for the full project map.

---

## References and Further Reading

- IAB Tech Lab: "Attribution and Conversion Measurement Guidelines" — foundational standard for digital attribution
- MRC: "Cross-Media Audience Measurement Standards" — view-through attribution methodology and holdout requirements
- Nielsen / Circana (formerly IRI): "Pharma Attribution Methodology" — script lift measurement and panel-based attribution for healthcare
- Geopath / Placer.ai: Point-of-care DOOH measurement documentation — mobile device pairing and location-graph attribution
- IAB Tech Lab: "Data Transparency Standards" — identity match quality disclosure requirements for attribution reporting
- The Consensus Layer Artifact 03: Frequency Capping — attribution joins depend on the same household identity resolution infrastructure as household-level frequency enforcement; the two systems share a schema foundation
- The Consensus Layer Artifact 05: Clean Rooms and PETs — view-through attribution in healthcare environments must be conducted in a privacy-preserving computation layer; raw impression-to-conversion joins may implicate PHI depending on venue type and advertiser category

---

*The Consensus Layer is a living reference built by Pete Duhon. Born in AdTech. Built for product teams translating business logic into technical infrastructure.*
*GitHub: github.com/peterduhon | Last updated: June 2026*
