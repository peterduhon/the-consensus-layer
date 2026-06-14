# The Consensus Layer

**Born in AdTech. Built for product teams translating business logic into technical infrastructure.**

*A living reference system for senior product managers who sit at the boundary between engineering capability and business outcome: the gap where the most expensive decisions get made with the least shared language.*

---

## What This Is

The Consensus Layer is a structured knowledge base built from direct observation of a recurring pattern across CTV, DOOH, and enterprise platform environments: engineering teams with strong technical depth operating without the business context needed to make commercially sound architecture decisions.

This is not a criticism of engineers. It is a structural diagnosis. CTV and DOOH platforms industrialized faster than domain knowledge could transfer. Engineers hired from platform, streaming, and IoT backgrounds were asked to build adtech infrastructure without the programmatic vocabulary that defines what clients actually need from that infrastructure. The result: log schemas that look complete until a client tries to reconcile a bill; measurement pipelines that report delivery without proof of audience; SLA commitments that do not exist for billion-dollar contracts.

The Consensus Layer addresses this by making the implicit explicit: the business logic that product managers carry in their heads, written down in a form that engineers can read, reference, and apply at the point of decision.

**It is not training material.** It is a decision reference: the document you send before a schema discussion, the framework you use to scope a feature, the shared vocabulary that makes the first meeting shorter.

---

## The Problem This Solves

When a CTV engineer builds a log schema without understanding that `bid_price`, `clearing_price`, and `win_price` are three different fields with three different commercial implications, the client builds a shadow reconciliation system. When a DOOH engineer treats a screen-on event as a billable impression without audience estimation, the advertiser applies a 40% discount to all future buys. When a platform team commits to 99.9% uptime without understanding what downtime costs a $10M quarterly contract, the SLA negotiation starts from the wrong number.

These are not engineering failures. They are **translation failures**: the business requirement was never communicated in terms the engineering decision could absorb.

The Consensus Layer is the translation layer.

---

## Project Architecture

The project is organized into four chapters, each addressing a different layer of the product-engineering translation problem. Chapter 1 is live and actively expanding. Chapters 2 through 4 are in active development.

```
the-consensus-layer/
│
├── README.md                          ← You are here
│
├── chapter-1-adtech-translation/      ← Live
│   ├── 01-first-price-second-price.md
│   ├── 02-viewability.md
│   ├── 03-frequency-capping.md        ← In progress
│   ├── 04-attribution-window.md       ← Planned
│   ├── 05-clean-rooms-pets.md         ← Planned
│   └── 06-sla-uptime.md               ← Planned
│
├── chapter-2-design-intelligence/     ← In development
│   ├── rfc-before-meeting.md
│   ├── two-layer-monitoring.md
│   └── toggle-driven-releases.md
│
├── chapter-3-api-first/               ← Planned
│   ├── openapi-before-jira.md
│   └── openapi-mcp-bridge.md
│
└── chapter-4-enterprise-playbook/     ← Accumulating
    └── (in progress)
```

---

## Chapter 1: AdTech Translation

The six concepts where business logic and engineering implementation most frequently diverge in CTV and DOOH environments. Each artifact maps a business concept to its technical implementation and explains the failure mode when the two are misaligned.

| # | Concept | Technical Focus | Status |
|---|---------|-----------------|--------|
| 01 | [First-Price vs. Second-Price Auction](./chapter-1-adtech-translation/01-first-price-second-price.md) | Bid shading; floor price optimization; log schema price fields | ✅ Published |
| 02 | [Viewability (MRC / IAS / MOAT)](./chapter-1-adtech-translation/02-viewability.md) | Pixel firing; viewport detection; SSAI completion beacons; DOOH OTS | ✅ Published |
| 03 | Frequency Capping | User/device graph deduplication; cross-device session management; household-level caps | 🔄 In progress |
| 04 | Attribution Window | Conversion pixel timing; post-click vs. post-view logic; long consideration cycles | 📋 Planned |
| 05 | Clean Rooms and PETs | Differential privacy; k-anonymity; query logging; privacy-preserving computation | 📋 Planned |
| 06 | SLA and Uptime Guarantees | Error budgets; circuit breakers; graceful degradation; contract-level implications | 📋 Planned |

**Format for each artifact:**

Every Chapter 1 document follows a consistent structure: The Business Problem; What Engineers Think (and why, given their prior context); What the Business Actually Needs; Technical Implementation (with working SQL and Python); Failure Mode (the commercial consequence of getting it wrong); Translation Conversation: Product and Engineering (how to explain it in 60 seconds without losing the room); Open-Source Implementation (connections to working code in the portfolio repos).

---

## Chapter 2: Design Intelligence

The operational patterns that separate product managers who ship features from product managers who ship systems. These are not frameworks borrowed from textbooks; they are practices that emerged from managing high-stakes technical decisions across organizations where engineering and business rarely spoke the same language.

| Artifact | What It Covers | Status |
|----------|----------------|--------|
| RFC Before the Meeting | How to distribute a Request for Comments 48 hours before any significant technical decision, define the question precisely, and create a record of what was known, what was disputed, and what was decided | 🔄 In development |
| Two-Layer Monitoring | The pre-release instrumentation layer (what you will measure to know if the release is healthy) and the post-release review layer (what you will analyze to understand whether the feature solved the right problem) | 🔄 In development |
| Toggle-Driven Releases | Feature flags and kill switches as a release discipline: no production deployment without a rollback mechanism; how to define the flag logic, who holds the toggle, and what the rollback trigger is | 🔄 In development |

**Origin:** These practices were developed and applied during Project Truth Engine at Kroger/84.51° (2023 to 2024), a platform reliability initiative that required building consensus across interdisciplinary teams with competing priorities before any feature entered the development queue. The RFC-before-meeting pattern specifically emerged from the challenge of advocating for platform maintenance work against feature roadmap pressure: a structured pre-meeting document changed the dynamic from advocacy to analysis.

---

## Chapter 3: API-First Infrastructure

The product methodology for teams building platforms that other engineers build on. API-first is not a technical preference; it is a product discipline that determines whether the platform is an asset or a dependency.

| Artifact | What It Covers | Status |
|----------|----------------|--------|
| OpenAPI Before Jira | Why the API contract should be written and reviewed before the Jira ticket is created: the contract is the spec, and the ticket is the execution record; how to use mock servers to enable parallel front-end and back-end development | 📋 Planned |
| OpenAPI and MCP Bridge | How OpenAPI specifications connect to Model Context Protocol (MCP) server definitions; the emerging pattern for AI-accessible product infrastructure | 📋 Planned |

**Context:** The OpenAPI-before-Jira practice emerged from observing a recurring failure mode: engineering teams building API endpoints to match internal implementation assumptions rather than external client requirements. The contract written before the ticket is written from the client's perspective; the contract written after is written from the builder's perspective. These are different documents.

---

## Chapter 4: Enterprise Product Playbook

The meta-layer: the practices that govern how product decisions get made and documented in organizations where the cost of a wrong decision is measured in contract renewals, not sprint velocity. This chapter accumulates as patterns emerge from the other three.

Topics in development include: building and maintaining a CYA documentation discipline that protects both the product team and the engineering team when decisions are later questioned; constructing a P&L visibility model that maps pipeline compute cost to client revenue so that product prioritization is grounded in margin, not instinct; and the cross-functional consensus framework for decisions that require alignment across VP-level stakeholders who do not share a reporting structure.

---

## Related Portfolio Repositories

The Consensus Layer is a knowledge system; the repositories below are the working implementations. Each one instantiates one or more concepts documented in the chapters above.

| Repository | What It Demonstrates | Chapter 1 Connection |
|------------|---------------------|----------------------|
| [BidOptimizerPro](https://github.com/peterduhon/BidOptimizerPro) | ML-based bid simulation and optimization; SQLite bid data analysis | Artifact 01: bid_price / clearing_price / win_price schema; floor price optimization |
| [Intent Segments / Signal CTV](https://github.com/peterduhon/intentsegments) | K-Means audience clustering; CTV signal ingestion; first-party data enrichment; yield simulation | Artifact 02: SSAI completion rate as viewability proxy; Artifact 05: clean room architecture |
| [Ad Tech SQL Analysis](https://github.com/peterduhon/SQL-Analysis) | SQLite bid data generation; SSP win rate analysis; advertiser spend queries | Artifact 01: floor price effectiveness SQL; Artifact 02: completion rate by segment SQL |
| [AdInsight: Reddit Ad Dashboard](https://github.com/peterduhon/adinsight) | Streamlit dashboard; SQLite integration; campaign performance visualization | Artifact 06: reporting layer as a client-facing product surface |
| [AdVantageX RTB Simulation](https://github.com/peterduhon/rtb-practice) | OpenRTB bid request/response simulation; auction analysis; daily reporting | Artifact 01: RTB auction mechanics; first-price vs. second-price clearing logic |
| [Performance Optimization in AdTech Bidding](https://github.com/peterduhon/performance-optimization) | Multithreaded bid simulation; execution time profiling; latency visualization | Artifact 06: latency as an SLA input; pipeline performance as a client retention factor |

These repositories are designed to be extended as The Consensus Layer evolves. When an artifact in Chapter 1 documents a concept, the corresponding repo gains a milestone: the code implements what the document explains.

---

## How to Use This

**If you are a product manager** joining a CTV or DOOH platform team: start with Chapter 1. Read the artifact for the concept that is most actively being debated on your team. Use the Translation Conversation section as a starting point for the next engineering discussion.

**If you are an engineer** working on a CTV or DOOH data platform: the Failure Mode section of each artifact describes the commercial consequence of the architecture decision you are currently making. It is not prescriptive; it is context. The schema decision is still yours.

**If you are evaluating this repository** as part of a hiring process: the project is intentionally incomplete. A project with a clear architecture and two published artifacts signals more than a project with twelve artifacts and no through-line. The commit history is the work log. The planned artifacts are the roadmap. The connection between the knowledge documents and the code repositories is the proof that this is a system, not a collection.

---

## About

Built by **Pete Duhon**, Senior Technical Product Manager with 10+ years across adtech, retail media, and platform product.

Prior work includes: Lead Technical Product Manager at Kroger/84.51° (Project Truth Engine and Product Athena; Snowflake; Grafana; GenAI LLM use cases); Technical Product Manager at The Wall Street Journal (MOAT; IVT mitigation; Permutive; privacy compliance across seven global brands; 100M impressions annually); Cloud Architect and Product Manager at Hudson's Bay Company (AI-powered DAM system; 80% workflow acceleration; 50% cost reduction).

Independent work includes blockchain Oracle protocol development (xLiquidity; Tellor Oracle award) and AI/ML integration consulting.

**GitHub:** [github.com/peterduhon](https://github.com/peterduhon)
**LinkedIn:** [Peter Duhon](https://www.linkedin.com/in/pete-duhon-7344765)
**Contact:** peterduhon[@]gmail.com

---

*The Consensus Layer is a living project. Artifacts are published as they are ready; the project is designed to be read as a work in progress, not a finished product. Last updated: June 2026.*
