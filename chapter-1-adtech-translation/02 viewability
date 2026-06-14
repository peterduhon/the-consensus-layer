# 02: Viewability (MRC / IAS / MOAT)

**The Consensus Layer: Chapter 1 — AdTech Translation**
*A shared language for product teams translating business logic into technical infrastructure.*

---

## The Business Problem

Advertisers pay for ads that are seen. Publishers are paid for ads that are delivered. These are not the same thing, and the gap between them is where billions of dollars of trust erode annually. Viewability is the industry's attempt to define and enforce the minimum condition for a delivered impression to count as a seen one. When engineers build delivery systems without understanding that definition, they build platforms that report high delivery numbers advertisers refuse to pay for.

---

## What Engineers Think

"The screen is on. The ad played. That's an impression."

Engineers from IoT and device infrastructure backgrounds, particularly those entering DOOH environments, often model delivery as a binary: the screen was powered, the content was sent, the impression occurred. Engineers entering CTV from video platform backgrounds apply a similar logic: the VAST tag fired, the ad loaded, the log recorded it.

The root cause is again a context mismatch. Engineers who have built infrastructure for internal content delivery, smart device networks, or video streaming platforms have operated in systems where delivery equals success: the packet arrived, the file rendered, the event fired. In those systems, the endpoint is also the audience. In advertising, the endpoint (the screen, the browser, the TV) is merely the proxy for a human being who may or may not have been present, attentive, or in-view. Delivery is necessary but not sufficient. Viewability is the bridge condition between delivery and value.

The downstream consequence: platforms that report delivered impressions without viewability signal give advertisers no way to distinguish premium, in-view inventory from wasted spend. Advertisers who cannot verify viewability will either discount the CPM they are willing to pay, demand third-party verification before renewing, or move budget to platforms where viewability is a first-class data field.

---

## What the Business Actually Needs

**The MRC Standard (what the industry agreed on for display and web video):**

The Media Rating Council defines a viewable impression as: 50% of the ad's pixels in view for a minimum of 1 continuous second for display, or 2 continuous seconds for video. This is the floor; the minimum condition for an impression to count. It is not a quality standard. It is a validity gate.

**Why the MRC standard does not translate to CTV:**

CTV ads play full-screen on a television set. There is no scroll, no tab switching, no fold. The entire creative is always 100% in view by definition; applying the 50%-for-2-seconds standard to CTV is trivially true and therefore meaningless as a quality signal. The MRC acknowledged this and issued updated guidance: for CTV, the industry-recognized proxy for viewability is **ad completion rate**, specifically the percentage of ads played to 100% as measured by SSAI quartile beacon events.

**Why the MRC standard does not translate to DOOH:**

Digital out-of-home screens (billboards, transit displays, venue screens) exist in physical space. There is no pixel viewport to measure, no scroll event to detect, no browser to instrument. The screen being powered and displaying content does not mean a human was present, facing the screen, or within a meaningful viewing distance. The DOOH industry uses a different proxy: **opportunity to see (OTS)**, derived from foot traffic data, camera-based audience measurement, or sensor-based dwell time. This is probabilistic by nature, not deterministic.

**Why this matters in 2024 to 2026:**

Both CTV and DOOH are channels that have grown rapidly by selling the promise of addressable, measurable audiences. Advertisers who cut their teeth on web display with MRC-standard viewability reporting are now buying CTV and DOOH expecting equivalent or superior measurement. The channels that can provide auditable, third-party-verified measurement will command premium CPMs; the channels that cannot will face discount pressure and budget reallocation. This is the measurement gap CTV and DOOH engineering teams are now being asked to close at the data layer.

---

## Technical Implementation

### The Three Measurement Layers and What Each Produces

| Layer | Web Display | CTV | DOOH |
|-------|-------------|-----|------|
| **Delivery signal** | Pixel fires on ad load | VAST impression beacon | Screen-on event; content playback confirmation |
| **Viewability signal** | JS viewport detection (IAS, MOAT, DoubleVerify) | SSAI completion quartile beacons (Q1/Q2/Q3/Q4) | Foot traffic sensors; camera-based dwell time; OTS model |
| **Fraud / IVT signal** | Bot detection; traffic quality scoring | Device fraud detection; app bundle spoofing detection | Physical location verification; duplicate play detection |
| **Third-party audit** | IAS, MOAT, DoubleVerify (pixel-based) | OMID (Open Measurement SDK); IAB CTV measurement guidelines | Geopath (OOH), Placer.ai; venue-specific sensor data |

### How Viewability Measurement Works in Web Environments (the reference model)

Web viewability measurement is JavaScript-based. A verification vendor (IAS, MOAT, DoubleVerify) wraps the ad creative in a measurement tag. When the ad loads, the tag executes viewport detection logic: it calculates what percentage of the ad's pixel area is within the visible browser window, and starts a timer when the threshold is met. The result is a binary: viewable or not viewable, reported at the impression level.

```python
# Illustrative model of web viewability scoring logic
# Production implementations are JavaScript-based in-browser; this is a Python
# representation of the decision logic for educational purposes

def is_viewable(pixels_in_view: float, total_pixels: float,
                seconds_in_view: float, ad_type: str) -> bool:
    """
    Determine if an impression meets MRC viewability standard.

    pixels_in_view:  number of the ad's pixels currently within the browser viewport
    total_pixels:    total pixel area of the ad creative
    seconds_in_view: continuous seconds the threshold has been maintained
    ad_type:         'display' or 'video' (different time thresholds apply)
    """
    pixel_threshold = 0.50          # MRC: 50% of pixels must be in view
    time_threshold_display = 1.0    # MRC: 1 continuous second for display
    time_threshold_video = 2.0      # MRC: 2 continuous seconds for video

    pixel_ratio = pixels_in_view / total_pixels
    if pixel_ratio < pixel_threshold:
        return False

    time_threshold = time_threshold_video if ad_type == 'video' else time_threshold_display
    return seconds_in_view >= time_threshold


# What the log schema needs to carry for viewability at the impression level:
#   - viewable (bool): did this impression meet the MRC threshold?
#   - time_in_view_seconds (float): how long were the threshold pixels in view?
#   - pixel_percentage_in_view (float): peak pixel-in-view percentage during the impression
#   - viewability_vendor (string): IAS, MOAT, DoubleVerify — which vendor measured this?
#   - measurement_method (string): 'client_side_js', 'server_side_omid', 'estimated'
```

### How Viewability Is Proxied in CTV: SSAI Completion Beacons

In CTV, JavaScript cannot execute in a smart TV app. The client-side pixel approach that powers web viewability measurement is not available. The measurement layer is the VAST beacon system embedded in the ad delivery pipeline.

VAST 4.x defines standardized tracking event URLs that fire at specific playback milestones. The SSAI system fires these beacons as the ad plays server-side, and the log records each event. Completion rate, derived from Q4 (100% completion) beacon firings, is the CTV industry's accepted viewability proxy.

```sql
-- CTV Completion Rate Analysis by Inventory Segment
-- This is the CTV equivalent of a web viewability report
-- Written for Snowflake syntax
-- BigQuery: replace DATEADD with DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
-- Redshift/Postgres: DATEADD syntax compatible; adjust window functions as needed

SELECT
    content_genre,
    device_type,
    pod_position,
    COUNT(*)                                                                AS total_impressions,

    -- Quartile completion rates (VAST beacon events)
    ROUND(SUM(CASE WHEN q1_fired  = TRUE THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS q1_rate_pct,
    ROUND(SUM(CASE WHEN q2_fired  = TRUE THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS q2_rate_pct,
    ROUND(SUM(CASE WHEN q3_fired  = TRUE THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS q3_rate_pct,
    ROUND(SUM(CASE WHEN q4_fired  = TRUE THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS completion_rate_pct,

    -- Error rate: impressions where the ad started but did not complete
    ROUND(SUM(CASE WHEN impression_fired = TRUE AND q4_fired = FALSE THEN 1 ELSE 0 END)
          * 100.0 / NULLIF(SUM(CASE WHEN impression_fired = TRUE THEN 1 ELSE 0 END), 0), 2)
                                                                           AS mid_ad_error_rate_pct,

    -- Average CPM for completed vs non-completed impressions
    ROUND(AVG(CASE WHEN q4_fired = TRUE THEN win_price END), 4)            AS avg_cpm_completed,
    ROUND(AVG(CASE WHEN q4_fired = FALSE THEN win_price END), 4)           AS avg_cpm_not_completed

FROM ctv_impression_log
WHERE event_date >= DATEADD(day, -30, CURRENT_DATE)
  AND inventory_type = 'CTV'

GROUP BY content_genre, device_type, pod_position
ORDER BY completion_rate_pct DESC;
```

**What this query produces that a client actually needs:**

Completion rate broken out by content genre, device type, and pod position gives the buyer three things: proof of delivery quality, a signal for where to concentrate budget, and the data to defend the CPM they paid in their own reporting to their client. An aggregate completion rate number without those dimensions is insufficient for a professional DSP buyer.

### DOOH: Opportunity to See (OTS) — the Probabilistic Model

DOOH measurement cannot rely on pixels or beacon events. The measurement model is audience estimation: given what we know about traffic at this location and time, how many people likely had an opportunity to see this ad?

```python
# Illustrative OTS estimation model for DOOH impressions
# Production systems use sensor data, camera analytics, or panel-calibrated traffic models
# PEDAGOGICAL SIMPLIFICATION: production systems apply audience measurement
# methodologies validated against panel data and certified by Geopath or equivalent

def estimate_ots(
    foot_traffic_count: int,
    screen_dwell_zone_radius_meters: float,
    avg_dwell_time_seconds: float,
    ad_duration_seconds: float,
    share_of_voice: float
) -> dict:
    """
    Estimate Opportunity to See for a DOOH impression.

    foot_traffic_count:          pedestrians or vehicles passing in the measurement window
    screen_dwell_zone_radius:    distance within which the screen is legibly visible
    avg_dwell_time_seconds:      average time a passerby spends within the dwell zone
    ad_duration_seconds:         length of the ad creative
    share_of_voice:              fraction of the loop this ad occupies (e.g. 0.25 = 1 in 4)
    """
    # Fraction of passersby who were present long enough to see the ad
    # If avg dwell >= ad duration, all counted traffic had OTS; otherwise scale proportionally
    exposure_probability = min(avg_dwell_time_seconds / ad_duration_seconds, 1.0)

    # OTS: people who passed * probability they were present during this ad's slot
    raw_ots = foot_traffic_count * exposure_probability * share_of_voice

    return {
        'estimated_ots': round(raw_ots),
        'exposure_probability': round(exposure_probability, 3),
        'measurement_method': 'probabilistic_dwell_model',
        'confidence': 'estimated'   # Never claim deterministic certainty for OTS
    }

# What the log schema must carry for DOOH measurement credibility:
#   - ots_estimated (int): the calculated audience estimate for this play
#   - measurement_source (string): 'sensor', 'camera', 'panel_calibrated', 'modeled'
#   - dwell_zone_radius_m (float): the physical measurement radius used
#   - foot_traffic_raw (int): the underlying traffic count before OTS adjustment
#   - measurement_vendor (string): Geopath, Placer.ai, venue-specific
#   - confidence_level (string): 'high', 'medium', 'estimated'
```

### Invalid Traffic in CTV and DOOH: Different Threat Vectors

Web IVT is primarily bot traffic: automated scripts that simulate user behavior to generate fake impressions. The MRC-accredited vendors (IAS, MOAT, DoubleVerify) detect this through behavioral analysis, IP reputation scoring, and device fingerprinting in JavaScript.

CTV IVT is structurally different. The primary fraud vectors are:

**App bundle spoofing:** A fraudulent app declares itself as a premium CTV app (e.g., a major streaming service) in the bid request. The advertiser pays premium CTV CPMs for an impression that may have run on a low-quality device or not run at all. Detection requires bid request validation against authorized seller lists (sellers.json, app-ads.txt for CTV).

**Device spoofing:** A server generates fake device IDs and simulates streaming activity to accumulate impressions. Detection requires cross-referencing device ID patterns against known device graph data and flagging statistical anomalies (e.g., a device ID that generates 10,000 impressions in an hour).

DOOH IVT is simpler in some ways and harder in others. A screen that loops an ad 500 times per day generates 500 logged plays. Whether any human saw any of those plays is the fraud question; duplicate physical play events are easy to detect; absent human audiences are not.

---

## Failure Mode

**What breaks when engineers treat "screen on" as "ad seen":**

In DOOH environments, a log schema that records every content play as a billable impression without audience estimation creates a structural transparency failure. Advertisers running campaigns across web, CTV, and DOOH cannot compare inventory quality across channels if the DOOH channel reports raw play counts while the other channels report audience-adjusted metrics. The result: DOOH looks cheap on a cost-per-impression basis and expensive on a cost-per-verified-audience basis. Sophisticated buyers learn this quickly and apply steep discounts; or they require third-party measurement certification before allocating meaningful budget.

In CTV environments, the equivalent failure is platforms that report impression-fired counts without completion rate. A DSP buying CTV inventory needs to prove to their advertiser client that the video ad completed. An impression without a completion event is a delivery claim, not a measurement claim. Platforms that do not surface Q4 beacon data, or that conflate impression beacons with completion beacons in the log schema, force buyers to choose between trusting the platform's unverifiable numbers or investing in their own parallel measurement layer. When they build the parallel layer, the platform has again become a commodity pipe.

This pattern has been observed across multiple CTV and DOOH platform migrations as these channels have industrialized programmatic buying. The engineering response to "viewability doesn't apply here" is technically correct but commercially dangerous: the buyer still needs an equivalent quality signal, and if the platform does not provide one, the buyer will find or build one.

---

## Translation Conversation: Product and Engineering

*How to bring a platform engineer from "screen on = impression" to "measurement is the product":*

"Imagine you're a restaurant and you charge by the plate. A delivered plate means food left the kitchen. But the customer's definition of a delivered meal is food that arrived at the table, was edible, and was eaten. For display ads on the web, the industry decided the minimum condition for 'edible' is that the ad was at least 50% visible for at least one second. For CTV, since the ad is always full screen, the equivalent condition is that the video played all the way through. For a DOOH screen in a subway station, the equivalent is that a human being was standing close enough to read it when it played. Our log schema needs to record not just 'the plate left the kitchen' but 'the plate was eaten.' Those are different events and they require different data fields."

The engineering implication to surface in that conversation: completion beacons and impression beacons are not interchangeable fields. A schema that uses a single `event_type` field and treats `impression` and `complete` as equivalent delivery signals will produce billing disputes the moment a client runs their own measurement vendor against the platform's numbers.

---

## Open-Source Implementation

These concepts connect directly to existing work in The Consensus Layer portfolio.

**Intent Segments / Signal CTV** (`github.com/peterduhon/intentsegments`) already models CTV audience segmentation using SSAI ad completion events as a first-class signal input. The `completion_rate` field in the Signal CTV schema extension is precisely the viewability proxy described in this document: Premium Streamers are defined in part by a completion rate threshold of 88% or above. This artifact provides the business reasoning for why that threshold exists and what it means commercially.

**Ad Tech SQL Analysis** (`github.com/peterduhon/SQL-Analysis`) provides the analytical foundation. The completion rate query in this document is the natural successor to the win rate by SSP queries already in that repo: once you know what cleared, you need to know whether it completed.

Both repos are designed to be extended as the concepts in The Consensus Layer evolve.

---

## Chapter 1: AdTech Translation — Full Series

| # | Concept | Technical Focus | Status |
|---|---------|-----------------|--------|
| 01 | First-Price vs. Second-Price Auction | Bid shading; floor price optimization; log schema price fields | ✅ Published |
| **02** | **Viewability (MRC / IAS / MOAT)** | Pixel firing; viewport detection; fraud filtering; CTV completion rate; DOOH OTS | ✅ Published |
| 03 | Frequency Capping | User/device graph deduplication; cross-device session management; household-level caps | 📋 Planned |
| 04 | Attribution Window | Conversion pixel timing; post-click vs. post-view logic; pharma consideration cycles | 📋 Planned |
| 05 | Clean Rooms and PETs | Differential privacy; k-anonymity; query logging; enabling queries without exposing raw data | 📋 Planned |
| 06 | SLA and Uptime Guarantees | Error budgets; circuit breakers; graceful degradation; $100M contract implications | 📋 Planned |

Chapters 2 through 4 extend The Consensus Layer beyond adtech translation into design intelligence, API-first product methodology, and enterprise product infrastructure. See the [repository README](../README.md) for the full project map.

---

## References and Further Reading

- MRC (Media Rating Council): "Viewable Impression Measurement Guidelines" — the foundational standard
- MRC: "CTV/OTT Measurement Guidelines" — updated guidance on completion rate as the CTV proxy
- IAB Tech Lab: "Open Measurement SDK (OMID)" — the CTV-compatible measurement standard replacing JavaScript pixels
- IAB Tech Lab: VAST 4.x specification; quartile beacon event definitions (Q1/Q2/Q3/Q4)
- Geopath: "Out-of-Home Audience Measurement Methodology" — the OTS standard for DOOH
- IAB Tech Lab: "Identifier for Advertising (IFA) for CTV" — device ID standards and anti-fraud requirements
- The Consensus Layer: Artifact 01 (First-Price vs. Second-Price Auction) — completion rate and CPM are linked; a platform that cannot verify completion cannot defend its CPM

---

*The Consensus Layer is a living reference built by Pete Duhon. Born in AdTech. Built for product teams translating business logic into technical infrastructure.*
*GitHub: github.com/peterduhon | Last updated: June 2026*
