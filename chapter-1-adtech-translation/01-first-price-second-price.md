# 01: First-Price vs. Second-Price Auction

**The Consensus Layer: Chapter 1 — AdTech Translation**
*A shared language for product teams translating business logic into technical infrastructure.*

---

## The Business Problem

Publishers and advertisers are negotiating billions of dollars of media spend through automated auctions that clear in under 200 milliseconds. The price a buyer pays, and the revenue a publisher earns, are not the same number in every auction type. Getting this wrong means advertisers overpay, publishers under-earn, or both sides lose trust in the platform.

---

## What Engineers Think

"Price is price. The highest bidder wins and pays what they bid."

Engineers entering CTV and DOOH from platform and infrastructure backgrounds often model auction clearing as a simple `MAX(bid)` operation. This is accurate for first-price auctions but wrong for second-price auctions, and critically wrong when both auction types are running simultaneously on the same supply.

The root cause is not carelessness: it is a mismatch in prior experience. Engineers from platform backgrounds typically build auction systems for internal resource allocation; a single tenant, a single pricing model, a single reconciliation path. Adtech auctions are multi-tenant by design. The same impression can clear through different auction logic depending on deal type (Programmatic Guaranteed, PMP, open auction), and the same buyer may see different prices depending on their contract terms. The `MAX(bid)` mental model works cleanly for internal systems where price is a computed result. It fails in advertising marketplaces where price is a negotiated outcome shaped by deal structure, floor logic, and auction type simultaneously.

The downstream consequence: engineers building data pipelines conflate `bid_price`, `clearing_price`, and `win_price` into a single field. That single field breaks client reconciliation, billing accuracy, and campaign optimization at the point where it matters most: the log export a DSP client uses to run their own business.

---

## What the Business Actually Needs

**In a second-price (Vickrey) auction:** the highest bidder wins but pays $0.01 above the second-highest bid (or the floor price, whichever is higher). This design was intended to incentivize truthful bidding; advertisers bid their true value because overbidding does not raise their cost.

**In a first-price auction:** the winner pays exactly what they bid. This became the dominant CTV and programmatic standard around 2019 to 2021 as exchanges moved away from the opacity of second-price dynamics. It introduces bid shading as a critical strategy: advertisers intentionally bid below their true value to avoid paying more than necessary to win.

**Why both still coexist in 2024 to 2026:** Programmatic Guaranteed (PG) deals and private marketplace (PMP) deals often use first-price logic. Open auction remnant inventory may still use second-price or hybrid models. A single data pipeline serving clients who run across all three deal types must distinguish clearing logic per impression; not per campaign, not per day.

**Why this matters more now than it did in 2019:** First-price auctions became the programmatic standard around 2019 to 2021 for web display and video inventory. The CTV and DOOH channels are only now completing that same migration; moving from direct-sold, manually negotiated placements to programmatic pipes. Engineers building CTV and DOOH platforms in 2024 to 2026 are encountering first-price logic for the first time in their careers, in channels that have no legacy of it. They are not repeating a solved problem. They are solving it fresh, without the institutional memory that web programmatic teams accumulated over five years of transition.

**The signal loss dimension:** Bid shading models are trained on historical clearing price data. As third-party identifiers deprecate (cookies on web, IDFA on mobile, and the absence of any persistent identifier in CTV), the historical data that bid shading algorithms depend on becomes thinner and less reliable at the audience segment level. This makes clean room data and first-party identity resolution not just a privacy strategy but a bid optimization input: without identity-resolved audiences, buyers cannot build accurate clearing price distributions, and bid shading degrades to blunt approximation.

---

## Technical Implementation

### The Three Prices Every Log Schema Must Carry

| Field | Definition | Auction Type | Why It Matters |
|-------|------------|--------------|----------------|
| `bid_price` | What the buyer submitted | Both | Client input; what they intended to pay |
| `clearing_price` | What the auction determined as the winning threshold | Second-price | Not the same as bid_price; often $0.01 above second bid or floor |
| `win_price` | What the buyer is actually charged | Both | In first-price: equals bid_price. In second-price: equals clearing_price |
| `floor_price` | Minimum CPM the SSP will accept | Both | Loss reason when no bid clears; must be in loss notices |

**In a second-price auction:** `win_price = clearing_price < bid_price` (usually)
**In a first-price auction:** `win_price = bid_price`

Any log schema that stores only one of these fields is incomplete for a professional DSP buyer. Any schema that conflates `win_price` and `bid_price` is a billing transparency failure.

### Bid Shading: The First-Price Optimization Layer

Bid shading is the algorithmic response to first-price auctions. Since buyers now pay what they bid, the incentive shifts from "bid your true value" to "bid the minimum amount needed to win."

**How bid shading works (pedagogical model):**

```python
# PEDAGOGICAL SIMPLIFICATION: this illustrates the logical structure of bid shading.
# Production systems use Bayesian updating on historical clearing price distributions,
# often with gradient-boosted or neural models trained on millions of auction outcomes.
# The static multiplier below (0.75) does not reflect real implementation; it is here
# to make the optimization objective visible.

def shaded_bid(true_value: float, historical_clearing_mean: float, floor_price: float) -> float:
    """
    Shade a bid below true value while targeting a competitive clearing range.

    true_value:               max CPM the buyer is willing to pay
    historical_clearing_mean: observed average clearing price for this inventory segment
                              (in production: derived from win/loss logs with identity signal)
    floor_price:              minimum CPM the SSP will accept for this impression
    """
    # Production objective: bid just above the expected clearing price, not at true value.
    # The gap between true_value and historical_clearing_mean is the "shade space."
    # Production models optimize this gap using P(win) curves fit to clearing price distributions.

    # Naive approximation for illustration: shade halfway between floor and true value,
    # anchored toward the observed clearing mean.
    shade_target = (historical_clearing_mean + true_value) / 2

    # Never bid below floor: guaranteed loss, no data value
    return max(shade_target, floor_price + 0.01)


# What this function requires from the data pipeline to work at production quality:
#   - historical_clearing_mean per inventory segment (not global average)
#   - floor_price per auction (dynamic floors change impression by impression)
#   - win/loss outcomes with reason codes (outbid, floor not met, timeout)
#   - identity signal stability: if audience identity degrades (cookie loss, no IDFA in CTV),
#     segment-level clearing distributions become noisy and shading accuracy drops
```

**What engineers building bid shading actually need from the data pipeline:**

- Historical clearing prices at the inventory segment level (not just overall averages)
- Floor price transparency per auction (floor prices are often dynamic and publisher-set)
- Win/loss data with the reason code: outbid, floor not met, no fill, timeout
- Bid-to-clear ratio over time: the delta between what you bid and what you needed to bid to win

Without this data in the log, buyers cannot tune their bid shading models. Without tuned bid shading, buyers overspend in first-price environments. Overspending DSPs reduce budgets or switch supply sources.

### Floor Price Optimization: The Publisher Side

Floor prices are the publisher's lever in first-price auctions. Unlike second-price dynamics, where a publisher's floor simply filters out low bids without affecting the winning price, in first-price auctions, the floor directly compresses the spread between what a buyer bids and what the publisher earns.

**The optimization problem:**

A floor set too low: winner pays their full bid (good for publisher), but the floor did no work.
A floor set too high: impressions go unfilled. Revenue = $0 on that impression.
A floor set dynamically: SSP algorithms estimate the bid landscape in real time and set floors that maximize expected revenue across the fill rate / CPM tradeoff.

```sql
-- Analyzing floor price effectiveness: are we setting floors that maximize revenue?
-- Written for Snowflake syntax. Dialect adaptations noted inline.
-- BigQuery: replace DATEADD(day, -30, CURRENT_DATE) with DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
-- Redshift: DATEADD syntax is the same; SPLIT_PART is also supported
-- Postgres: replace DATEADD with CURRENT_DATE - INTERVAL '30 days'

SELECT
    floor_price_bucket,
    COUNT(*)                                          AS total_auctions,
    SUM(CASE WHEN win_price IS NOT NULL THEN 1 END)   AS filled_auctions,
    ROUND(
        SUM(CASE WHEN win_price IS NOT NULL THEN 1 END) * 100.0 / COUNT(*),
        2
    )                                                 AS fill_rate_pct,
    ROUND(AVG(win_price), 4)                          AS avg_win_price,
    ROUND(AVG(win_price) * (
        SUM(CASE WHEN win_price IS NOT NULL THEN 1 END) * 1.0 / COUNT(*)
    ), 4)                                             AS expected_revenue_per_auction

FROM (
    SELECT
        win_price,
        CASE
            WHEN floor_price < 5.00  THEN 'Under $5'
            WHEN floor_price < 10.00 THEN '$5 to $10'
            WHEN floor_price < 15.00 THEN '$10 to $15'
            WHEN floor_price < 20.00 THEN '$15 to $20'
            ELSE 'Over $20'
        END AS floor_price_bucket
    FROM impression_log
    WHERE event_date >= DATEADD(day, -30, CURRENT_DATE)
      AND inventory_type = 'CTV'
)

GROUP BY floor_price_bucket
ORDER BY CAST(SPLIT_PART(floor_price_bucket, '$', 1) AS FLOAT) NULLS LAST;
```

This query surfaces the expected revenue per auction by floor bucket; the publisher's tool for calibrating dynamic floor optimization.

---

## Failure Mode

**What breaks when engineers treat price as price:**

In CTV environments, clients who are DSPs themselves often receive raw log exports as the primary data product. When a log schema stores a single `price` field without distinguishing `bid_price`, `clearing_price`, and `floor_price`, the DSP client cannot:

1. Reconcile what they were charged against what they bid
2. Identify impressions where they won only because a competitor timed out (not because their bid was competitive)
3. Tune their bid shading algorithm against actual market clearing data
4. Prove to their own advertiser clients what clearing prices actually were

The platform-level consequence reaches further than a single client dispute. When the DSP client cannot trust the log, they build a shadow reconciliation system around it: their own pipeline, their own price calculations, their own assumptions about what the platform charged. At that point, the platform has not just lost a billing conversation; it has lost the narrative that its data is reliable. A $10M quarterly contract does not renew on performance alone when the client is running a parallel data operation to verify numbers the platform should have provided. The platform becomes a commodity pipe. The client begins evaluating alternatives.

This pattern has been observed across multiple CTV and DOOH platform migrations as these channels have moved from direct-sold to programmatic. The underlying cause is consistent: engineering teams build log pipelines optimized for internal monitoring rather than external client use cases. The schema that works for debugging is not the schema that works for a DSP's reconciliation team.

---

## Translation Conversation: Product and Engineering

*How to bring a platform engineer into this distinction without losing them in auction theory:*

"Think about eBay. Classic eBay is second-price: you bid your max, the auction engine bids the minimum needed to beat the next person, and you usually pay less than your max. That's how Google's ad auctions worked for years. Now think about a silent auction at a fundraiser: you write a number on a card, the highest card wins, and you pay exactly what you wrote. That's first-price. CTV programmatic made that shift around 2019 for web inventory; CTV and DOOH are making it now. The important difference for our data pipeline: in the silent auction model, we need to store what you wrote on the card, what the next person wrote, and what the minimum the event organizers would accept. Those are three different numbers. A log schema that stores one of them looks complete until a client tries to reconcile a bill."

The goal of this conversation is not to make the engineer an auction theory expert. It is to make the schema decision visible as a product decision with commercial consequences. Once an engineer understands that a single `price` field means the client builds a shadow system, the architecture conversation changes.

---

## Open-Source Implementation

These concepts are instantiated in working code rather than described in the abstract.

**BidOptimizerPro** (`github.com/peterduhon/BidOptimizerPro`) implements bid simulation and ML-based bid success prediction. The current version uses a unified price field in its synthetic data generation; the next milestone extends the schema to carry `bid_price`, `clearing_price`, and `win_price` as separate fields, with separate clearing logic paths for first-price and second-price auctions. This makes the repo a live artifact of the schema decision described in this document.

**Ad Tech SQL Analysis** (`github.com/peterduhon/SQL-Analysis`) implements win rate by SSP and advertiser spend analysis against a synthetic SQLite impression log. The floor price effectiveness query in this document is the natural next analytical layer in that suite; it sits one level above win rate and asks whether the floor strategy that produced those win rates was revenue-optimal.

Both repos are designed to be extended as the concepts in The Consensus Layer evolve.

---

## Chapter 1: AdTech Translation — Full Series

This document is part of The Consensus Layer, Chapter 1. The remaining artifacts in this chapter address the other core concepts where business logic and engineering implementation most frequently diverge in CTV and DOOH environments:

| # | Concept | Technical Focus | Status |
|---|---------|-----------------|--------|
| **01** | **First-Price vs. Second-Price Auction** | Bid shading; floor price optimization; log schema price fields | ✅ Published |
| 02 | Viewability (MRC / IAS / MOAT) | Pixel firing; viewport detection; fraud filtering; CTV completion rate as proxy | 🔄 In progress |
| 03 | Frequency Capping | User/device graph deduplication; cross-device session management; household-level caps | 📋 Planned |
| 04 | Attribution Window | Conversion pixel timing; post-click vs. post-view logic; pharma consideration cycles | 📋 Planned |
| 05 | Clean Rooms and PETs | Differential privacy; k-anonymity; query logging; enabling queries without exposing raw data | 📋 Planned |
| 06 | SLA and Uptime Guarantees | Error budgets; circuit breakers; graceful degradation; $100M contract implications | 📋 Planned |

Chapters 2 through 4 extend The Consensus Layer beyond adtech translation into design intelligence, API-first product methodology, and enterprise product infrastructure. See the [repository README](../README.md) for the full project map.

---

## References and Further Reading

- IAB Tech Lab OpenRTB 2.6 Specification: `bid_price`, `clearing_price`, and `loss_reason` field definitions
- Index Exchange Whitepaper: "The Mechanics of First-Price Auctions in Programmatic Advertising" (2019)
- Prebid.org documentation on bid shading integration for Header Bidding
- IAB Tech Lab: VAST 4.x specification; beacon event taxonomy for CTV delivery measurement
- The Consensus Layer: `AdTech_CTV_Translation_Guide` (win price vs. clearing price vs. floor price schema requirements for CTV log delivery)

---

*The Consensus Layer is a living reference built by Pete Duhon. Born in AdTech. Built for product teams translating business logic into technical infrastructure.*
*GitHub: github.com/peterduhon | Last updated: June 2026*
