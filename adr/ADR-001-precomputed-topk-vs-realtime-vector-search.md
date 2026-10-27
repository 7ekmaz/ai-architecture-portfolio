# ADR-001: Precomputed top-K + real-time re-rank for homepage recommendations

- **Status:** Accepted
- **Date:** 2026-10-27
- **Use case:** Homepage recommendations for a returning user (Marketing Automation System)
- **Diagram:** [Homepage Recommendations Flow](../diagrams/Homepage%20Recommendations%20Flow.drawio)

## Context

A returning customer opens the homepage and should see 20 personalized products.

- **Latency:** P95 ≤ 150 ms server side.
- **Availability:** the homepage must render even when the ML components are down.
- **Traffic:** about 35 QPS on average, about 350 QPS at a Ramadan peak.
- **Data flow:** customer data reaches the analytics side through CDC → Kafka. The operational DB must not serve analytical reads.

## Options

| | **A. Precomputed top-K in CDP + real-time re-rank** | **B. Real-time Vector DB query per request** |
|---|---|---|
| How | Airflow batch-scores active users nightly or hourly and writes top-200 candidates per user to the CDP. At request time, the engine reads them plus online session features (Flink) and re-ranks to 20. | At request time, the engine builds a user embedding from features, queries the Vector DB for the top-200 similar items, then ranks them. |
| Latency (P95) | ~85 ms (key lookups + light re-rank) | ~135–165 ms; ANN tail latency rises with load |
| Cost | Compute once per user per night, off-peak | Vector DB + embedding model sized for peak QPS, running 24/7 |
| Operational DB load | None (CDC) | None (CDC) |
| Freshness | Candidates up to 1 h old; session features adjust the ranking | Reacts to the latest click |
| Failure mode | Fallback to popular-in-segment list from CDP | Vector DB down = no recommendations |

## Decision

**Option A.** It meets the P95 target with 65 ms of headroom, it has the lowest running cost, and it degrades gracefully.

## Consequences

- **Positive:**
  - The request path contains only key lookups.
  - Heavy compute runs off-peak.
  - The Vector DB is not needed for this use case.
- **Negative:**
  - Candidates can be up to 1 h stale.
  - New items only enter candidate lists after the next batch run.
- **Revisit when:** the catalog changes hourly, or an A/B test shows session-aware retrieval lifts CTR enough to justify the cost of Option B.
