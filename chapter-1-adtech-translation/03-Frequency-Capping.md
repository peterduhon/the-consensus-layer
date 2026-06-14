# 03: Frequency Capping

**The Consensus Layer: Chapter 1 — AdTech Translation**
*A shared language for product teams translating business logic into technical infrastructure.*

---

## The Business Problem

Every advertiser running a digital campaign has a number in their head: how many times is too many times for one person to see the same ad. Exceed that number and the ad becomes noise; the brand becomes annoying; the audience tunes out or, worse, develops negative associations. Fall below it and the campaign underdelivers against its awareness or recall objectives. Frequency capping is the mechanism that enforces the ceiling. When it works correctly, it is invisible. When it fails, the client's customers notice before the client does.

In web programmatic, frequency capping is a solved problem with a twenty-year implementation history. In CTV and cross-device environments, it is an active infrastructure challenge that most platforms are still resolving. The gap between "we have frequency capping" and "our frequency capping works at the household level across every device a family owns" is where campaign performance and client trust are won or lost.

---

## What Engineers Think

"Frequency capping is a counter. Every time a device sees the ad, increment the counter. When the counter hits the cap, stop serving."

Engineers entering CTV from platform and infrastructure backgrounds model frequency capping as a stateful rate limiter: a standard computer science problem with a well-understood solution set (Redis counters, sliding window algorithms, token buckets). This model is correct for the unit of identity it operates on. The problem is the unit of identity.

In web programmatic, the unit of identity is the user, approximated by a cookie or a hashed email address. One person, one identifier, one counter. The model is imperfect (users clear cookies, use multiple browsers) but it is directionally correct at scale.

In CTV, there is no cookie. The unit of identity available at the device level is the device ID: a Roku device ID (RIDA), a Fire TV identifier, an Apple TV vendor ID. These are stable within a device but they are device-level, not person-level and certainly not household-level. A household with a Roku in the living room, a Fire TV in the bedroom, and a smart TV in the kitchen has three separate device IDs. An engineer who builds frequency capping as a per-device counter will serve the same ad three times to the same household simultaneously, once on each screen, without registering a single frequency violation.

The problem inverts in DOOH. There, engineers often model frequency capping as a screen-level counter: the digital billboard showed the ad three times today, so the cap is met. But the advertiser's unit of identity is the audience member, not the screen. Without mobile device pairing, camera-based audience estimation, or location-graph inference, a screen-level counter cannot distinguish three different passersby from one person passing the same screen three times. The cap fires correctly at the screen level. The frequency objective is missed entirely at the audience level. The wrong unit of identity appears in both environments; it is just that CTV has too many identifiers per household while DOOH has no persistent identifier at all.

The downstream consequence is not just wasted impressions. It is the specific kind of brand damage that happens when a viewer sees the same thirty-second ad three times in one evening across three screens in their home. Advertisers track this. It surfaces in brand safety and sentiment data. It becomes a renewal conversation.

---

## What the Business Actually Needs

**The three levels of frequency enforcement and what each requires:**

| Level | Unit of Identity | What It Requires | Where It Fails Without It |
|-------|-----------------|------------------|--------------------------|
| **Device-level** | Single device ID (RIDA, Fire TV ID) | Simple counter per device ID per time window | Easy to implement; fails for multi-device households |
| **Household-level** | All devices at a shared IP or resolved via device graph | Device graph with household ID as the join key | Correct for CTV; fails without identity infrastructure |
| **Cross-channel** | Same household across CTV, web, and mobile | Authenticated identity or probabilistic graph spanning all channels | Requires first-party data or identity partner integration |

**Why household-level is the correct target for CTV in 2024 to 2026:**

Television is a household medium. The living room Roku is not one person's device; it is the household's device. Frequency capping that operates at the device level for CTV inventory is not just technically incomplete; it is wrong for the medium. An advertiser buying CTV inventory to reach households expects household-level frequency enforcement. Any platform that cannot deliver it is selling a lower-quality product than the advertiser believes they are buying.

**The cross-channel dimension:** Advertisers running campaigns across web, CTV, and mobile need their frequency cap to span all three. If a user has already seen the ad four times on web and mobile, they should not receive it again on their living room Roku regardless of which channel delivers next. This requires a unified identity layer: either authenticated first-party data (a logged-in user whose web identity can be matched to their CTV household) or a probabilistic device graph that resolves web cookies, mobile device IDs, and CTV device IDs to a shared household identifier.

**The privacy constraint:** Frequency capping at the household level requires knowing which devices belong to the same household. This knowledge is derived from deterministic signals (shared authenticated login) or probabilistic signals (shared IP address, co-viewing patterns, device graph inference). Both approaches have privacy implications that must be designed into the architecture: IP addresses are sensitive data under GDPR and CPRA; device graph data from third parties must be evaluated for consent signal compatibility.

---

## Technical Implementation

### Identity Resolution: The Foundation of Household Frequency Capping

Household-level frequency capping is only as good as the household resolution underneath it. The resolution problem has two components: which devices belong to the same household, and how confident the system is in that determination.

**Deterministic resolution (high confidence):**

The device belongs to the same household as another known device because a user authenticated on both. A viewer who logs into a streaming service on their Roku and also logs in on their phone provides a deterministic link: the platform knows, from first-party data, that these two devices are used by the same person (or household). Match rate for authenticated inventory is typically 68% or above for publishers with strong first-party data; see the Signal CTV PRD for the enrichment layer architecture.

**Probabilistic resolution (moderate confidence):**

Devices at the same IP address at similar times are likely in the same household. This is the dominant method for unauthenticated CTV inventory. It is effective in most residential contexts (a household's Roku, Fire TV, and smart TV all share the same home IP address) but degrades in shared living environments, apartment buildings with NAT-level IP sharing, and mobile devices that carry household IP only when on home Wi-Fi.

```python
# Household identity resolution: illustrative model
# Production systems use device graph providers (LiveRamp, Experian, TransUnion)
# and combine deterministic and probabilistic signals with confidence scoring.
# PEDAGOGICAL SIMPLIFICATION: this represents the decision logic, not a production graph.

from dataclasses import dataclass
from typing import Optional
from enum import Enum

class ResolutionMethod(Enum):
    DETERMINISTIC = "deterministic"   # Authenticated login match
    PROBABILISTIC = "probabilistic"   # IP + timestamp correlation
    GRAPH_INFERRED = "graph_inferred" # Third-party device graph resolution

@dataclass
class DeviceResolution:
    device_id: str
    household_id: str
    resolution_method: ResolutionMethod
    confidence_score: float           # 0.0 to 1.0
    ip_address: str
    last_seen_timestamp: str

def resolve_household(
    device_id: str,
    ip_address: str,
    auth_email_hash: Optional[str],
    device_graph_lookup_fn,           # External graph provider call
    confidence_threshold: float = 0.70
) -> Optional[DeviceResolution]:
    """
    Attempt to resolve a device ID to a household ID using available signals.

    Returns a DeviceResolution if confidence meets threshold; None otherwise.
    Low-confidence resolutions should NOT be used for frequency cap enforcement:
    a false household match creates over-capping (two households treated as one);
    a false non-match creates under-capping (one household treated as two).
    """

    # Step 1: Deterministic — authenticated email hash match
    if auth_email_hash:
        household_id = lookup_by_auth_hash(auth_email_hash)
        if household_id:
            return DeviceResolution(
                device_id=device_id,
                household_id=household_id,
                resolution_method=ResolutionMethod.DETERMINISTIC,
                confidence_score=0.98,
                ip_address=ip_address,
                last_seen_timestamp=current_timestamp()
            )

    # Step 2: Probabilistic — IP address correlation
    ip_household_candidates = lookup_by_ip(ip_address)
    if ip_household_candidates:
        best_candidate = max(ip_household_candidates, key=lambda c: c['confidence'])
        if best_candidate['confidence'] >= confidence_threshold:
            return DeviceResolution(
                device_id=device_id,
                household_id=best_candidate['household_id'],
                resolution_method=ResolutionMethod.PROBABILISTIC,
                confidence_score=best_candidate['confidence'],
                ip_address=ip_address,
                last_seen_timestamp=current_timestamp()
            )

    # Step 3: Third-party device graph
    graph_result = device_graph_lookup_fn(device_id)
    if graph_result and graph_result['confidence'] >= confidence_threshold:
        return DeviceResolution(
            device_id=device_id,
            household_id=graph_result['household_id'],
            resolution_method=ResolutionMethod.GRAPH_INFERRED,
            confidence_score=graph_result['confidence'],
            ip_address=ip_address,
            last_seen_timestamp=current_timestamp()
        )

    # Resolution failed or below threshold: treat as unresolved device
    # Enforcement decision: cap at device level to avoid under-capping;
    # flag in log as unresolved for audit purposes
    return None
```

### Frequency Enforcement: The Counter Logic

Once household identity is resolved, the frequency enforcement layer is a stateful counter with time-window management. The implementation choices here determine whether capping is enforced correctly across a distributed serving environment.

```python
# Frequency cap enforcement: Redis-backed sliding window counter
# This pattern is production-viable; exact Redis command syntax may vary by client library.
# The key design decisions: counter key structure, window granularity, and cap breach response.

import redis
import time
from typing import Tuple

class FrequencyCapEnforcer:
    """
    Enforces frequency caps at household level using Redis sorted sets
    as sliding window counters.

    Key structure: freq:{campaign_id}:{household_id}:{window_type}
    Window types: daily, weekly, flight (campaign lifetime)
    """

    def __init__(self, redis_client: redis.Redis):
        self.r = redis_client
        self.KEY_TTL_SECONDS = {
            'daily': 86400,
            'weekly': 604800,
            'flight': 2592000   # 30 days; adjust per campaign flight length
        }

    def check_and_increment(
        self,
        campaign_id: str,
        household_id: str,
        cap_limit: int,
        window_type: str = 'daily'
    ) -> Tuple[bool, int]:
        """
        Check whether this household has reached the frequency cap.
        If not, increment the counter and allow the impression.

        Returns: (allowed: bool, current_count: int)

        The atomic check-and-increment prevents race conditions in high-concurrency
        serving environments where multiple ad requests for the same household
        may arrive simultaneously.
        """
        now = time.time()
        window_seconds = self.KEY_TTL_SECONDS[window_type]
        window_start = now - window_seconds
        key = f"freq:{campaign_id}:{household_id}:{window_type}"

        pipe = self.r.pipeline()
        # Remove impressions outside the current window (sliding window)
        pipe.zremrangebyscore(key, '-inf', window_start)
        # Count impressions within the window
        pipe.zcard(key)
        results = pipe.execute()
        current_count = results[1]

        if current_count >= cap_limit:
            # Cap reached: do not serve, do not increment
            return False, current_count

        # Under cap: record this impression and set TTL
        pipe = self.r.pipeline()
        pipe.zadd(key, {str(now): now})
        pipe.expire(key, window_seconds)
        pipe.execute()

        return True, current_count + 1

    def get_household_frequency(
        self,
        campaign_id: str,
        household_id: str,
        window_type: str = 'daily'
    ) -> int:
        """
        Read current frequency for reporting and log enrichment.
        Called at log write time to enrich impression records with frequency context.
        """
        now = time.time()
        window_seconds = self.KEY_TTL_SECONDS[window_type]
        window_start = now - window_seconds
        key = f"freq:{campaign_id}:{household_id}:{window_type}"
        self.r.zremrangebyscore(key, '-inf', window_start)
        return self.r.zcard(key)
```

**What the log schema must carry for frequency cap auditability:**

```
household_id          (string):  the resolved household ID used for cap enforcement
household_resolution  (string):  'deterministic', 'probabilistic', 'graph_inferred', 'unresolved'
resolution_confidence (float):   0.0 to 1.0; low-confidence resolutions should be flagged
frequency_at_serve    (int):     the household's impression count at time of serving (1-indexed)
cap_limit             (int):     the campaign's configured frequency cap for this window
cap_window_type       (string):  'daily', 'weekly', 'flight'
device_id             (string):  the individual device ID (Roku RIDA, Fire TV, etc.)
device_count_in_hh    (int):     number of known devices in this household (for audit)
```

**Note on DOOH frequency schema:** The fields above are CTV-native; they assume a persistent device identifier that can be resolved to a household. In DOOH environments, none of these fields exist in the same form. The DOOH frequency schema primitive is different in kind: `screen_id` (the physical display), `play_count` (how many times the creative ran on that screen in the window), and `estimated_audience_per_play` (the probabilistic OTS count per play cycle, derived from foot traffic or camera estimation). DOOH frequency enforcement is not a cap on a counter per identity; it is a cap on plays per screen weighted by estimated audience exposure. These are architecturally distinct problems that happen to share a name.

### SQL: Frequency Distribution Analysis

Understanding whether frequency capping is working requires analyzing the actual distribution of impressions per household; not just whether the cap was technically enforced.

```sql
-- Household frequency distribution analysis
-- Are we serving the right number of times, or are some households seeing too many?
-- Written for Snowflake syntax
-- BigQuery: replace DATEADD with DATE_SUB(CURRENT_DATE, INTERVAL 7 DAY)
-- Redshift/Postgres: DATEADD syntax compatible

WITH household_frequency AS (
    SELECT
        campaign_id,
        household_id,
        household_resolution,
        COUNT(*)                                            AS impressions_served,
        COUNT(DISTINCT device_id)                          AS devices_reached_in_hh,
        MAX(frequency_at_serve)                            AS peak_frequency,
        MIN(event_timestamp)                               AS first_impression,
        MAX(event_timestamp)                               AS last_impression,
        AVG(resolution_confidence)                         AS avg_resolution_confidence

    FROM ctv_impression_log
    WHERE event_date >= DATEADD(day, -7, CURRENT_DATE)
      AND campaign_id = :campaign_id

    GROUP BY campaign_id, household_id, household_resolution
),

frequency_buckets AS (
    SELECT
        campaign_id,
        CASE
            WHEN impressions_served = 1  THEN '1x'
            WHEN impressions_served = 2  THEN '2x'
            WHEN impressions_served = 3  THEN '3x'
            WHEN impressions_served <= 5 THEN '4-5x'
            WHEN impressions_served <= 9 THEN '6-9x'
            ELSE '10x+'                                    -- Frequency cap breach or erosion
        END                                                AS frequency_bucket,
        household_resolution,
        COUNT(*)                                           AS household_count,
        SUM(impressions_served)                            AS total_impressions,
        ROUND(AVG(devices_reached_in_hh), 2)              AS avg_devices_per_hh,
        ROUND(AVG(avg_resolution_confidence), 3)          AS avg_id_confidence

    FROM household_frequency
    GROUP BY campaign_id, frequency_bucket, household_resolution
)

SELECT
    frequency_bucket,
    household_resolution,
    household_count,
    total_impressions,
    avg_devices_per_hh,
    avg_id_confidence,
    ROUND(household_count * 100.0 / SUM(household_count) OVER (PARTITION BY campaign_id), 2)
                                                           AS pct_of_households

FROM frequency_buckets
ORDER BY
    CASE frequency_bucket
        WHEN '1x'   THEN 1
        WHEN '2x'   THEN 2
        WHEN '3x'   THEN 3
        WHEN '4-5x' THEN 4
        WHEN '6-9x' THEN 5
        ELSE 6
    END,
    household_resolution;
```

**What this query surfaces:** Households in the `10x+` bucket with `household_resolution = 'unresolved'` are the most urgent finding; they represent devices that are not being captured under a household ID and are therefore receiving uncapped impressions. Households with high `avg_devices_per_hh` and `household_resolution = 'probabilistic'` or `'graph_inferred'` at low confidence scores are candidates for identity resolution review.

---

## Failure Mode

**What breaks when frequency capping operates at device level in a household medium:**

In CTV environments, a platform that enforces frequency caps at the device level rather than the household level will systematically overserve households that own multiple streaming devices; which, in 2024 to 2026, describes the majority of the addressable CTV audience. Research from multiple measurement vendors consistently shows that households with three or more connected devices receive 2 to 4 times the intended frequency for campaigns that enforce per-device caps.

The commercial consequences compound across three stakeholder relationships simultaneously. The viewer experiences ad fatigue: the same creative repeating across every screen in the home. The advertiser pays for impressions that are not delivering incremental reach; they are paying for repetition against an already-saturated household. The platform absorbs the complaint when the advertiser runs their own frequency analysis and discovers the distribution: households clustered at 8 to 12 impressions against a 3x daily cap.

At that point, the frequency cap failure is not a technical finding; it is a trust finding. The platform said the cap was enforced. The client's data shows it was not. The renewal conversation begins from a deficit.

Cross-channel frequency failures are harder to detect and more expensive when they surface. An advertiser running a 5x weekly cap across web, mobile, and CTV may find that their web and mobile campaigns consumed the cap before CTV had an opportunity to deliver. Or the inverse: CTV ran uncapped because the platform had no access to the web and mobile impression history. Either failure mode results in the same outcome: the campaign's effective frequency is either over or under target, and the client's measurement vendor will find it.

This pattern has been observed across multiple CTV and DOOH platform migrations. In CTV, the unit of identity is wrong: device instead of household. In DOOH, the unit of identity is absent: screen instead of audience member. The architecture decisions that follow from each error are different; the commercial consequence for the client is the same. The identity infrastructure required for household-level frequency capping in CTV is often scoped as a future roadmap item rather than a prerequisite; the decision is made because the cap technically fires on each device. The commercial consequence of that decision is deferred until the first client who runs a post-campaign frequency analysis.

---

## Translation Conversation: Product and Engineering

*How to move a platform engineer from per-device counter to household identity infrastructure:*

"Frequency capping on a single device is a rate limiter. We know how to build rate limiters; they're table stakes for any distributed system. The problem is that CTV is watched on televisions, and televisions are household devices, not personal devices. If a family has three streaming devices and we cap each one independently, we're not enforcing a frequency cap; we're enforcing three separate frequency caps that happen to be for the same ad. The household never hits the cap; they just get the ad once per screen per day.

The fix is not a harder rate limiter. It is a better unit of identity: the household ID instead of the device ID. That requires resolving which devices belong to the same household before we decide whether to serve. The infrastructure question is: what is our confidence in that resolution, and what do we do when we can't resolve it? Low-confidence resolutions that produce false household groupings will over-cap some households and under-cap others. The log needs to carry both the household ID and the resolution confidence so clients can audit our enforcement and we can improve the model."

---

## Open-Source Implementation

**Media Publisher Identity Graph Explorer** (`github.com/peterduhon/media-publisher-identity-graph-explorer`) is a direct instantiation of the household identity resolution problem described in this artifact. The interactive tool visualizes deterministic and probabilistic device connections (streaming login via email hash match; iOS device ID via authenticated login; Roku ID via IP and timestamp correlation; LiveRamp RampID via device graph inference) with confidence scores per connection. The resolution method taxonomy in this artifact (deterministic, probabilistic, graph-inferred) maps exactly to the connection types in that explorer.

**Intent Segments / Signal CTV** (`github.com/peterduhon/intentsegments`) implements household-level segmentation as a first-class schema primitive. The `household_id` join key across the `viewing_events`, `genre_scores`, and `email_matches` tables in the Signal CTV schema extension is the same household ID that frequency cap enforcement depends on. A platform that can resolve households for audience segmentation has the identity infrastructure needed for household-level frequency capping; the two capabilities share a foundation.

---

## Chapter 1: AdTech Translation — Full Series

| # | Concept | Technical Focus | Status |
|---|---------|-----------------|--------|
| 01 | [First-Price vs. Second-Price Auction](./01-first-price-second-price.md) | Bid shading; floor price optimization; log schema price fields | ✅ Published |
| 02 | [Viewability (MRC / IAS / MOAT)](./02-viewability.md) | Pixel firing; viewport detection; SSAI completion beacons; DOOH OTS | ✅ Published |
| **03** | **Frequency Capping** | User/device graph deduplication; cross-device session management; household-level caps | ✅ Published |
| 04 | Attribution Window | Conversion pixel timing; post-click vs. post-view logic; long consideration cycles | 📋 Planned |
| 05 | Clean Rooms and PETs | Differential privacy; k-anonymity; query logging; privacy-preserving computation | 📋 Planned |
| 06 | SLA and Uptime Guarantees | Error budgets; circuit breakers; graceful degradation; contract-level implications | 📋 Planned |

Chapters 2 through 4 extend The Consensus Layer beyond adtech translation into design intelligence, API-first product methodology, and enterprise product infrastructure. See the [repository README](../README.md) for the full project map.

---

## References and Further Reading

- IAB Tech Lab: "Identifier for Advertising (IFA) for Connected TV" — RIDA and device ID standards for CTV frequency management
- IAB Tech Lab: OpenRTB 2.6 specification; `ifa`, `ifa_type`, and `lmt` (limit ad tracking) fields in the device object
- LiveRamp: Identity Resolution documentation — deterministic vs. probabilistic matching methodology
- Prebid.org: User ID Module documentation — cross-environment identity resolution for web/CTV campaigns
- MRC: "Cross-Media Audience Measurement Standards" — guidance on frequency deduplication across channels
- The Consensus Layer Artifact 05: Clean Rooms and PETs — the privacy-preserving architecture for identity resolution without raw data exposure
- The Consensus Layer Artifact 01: First-Price vs. Second-Price Auction — household identity and bid shading share a dependency: both require device-to-household resolution to function correctly at the campaign level

---

*The Consensus Layer is a living reference built by Pete Duhon. Born in AdTech. Built for product teams translating business logic into technical infrastructure.*
*GitHub: github.com/peterduhon | Last updated: June 2026*
