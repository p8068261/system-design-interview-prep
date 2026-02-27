# System Design Interview: Multi-Tenant Real-Time Analytics Pipeline

## Question: Multi-Tenant Real-Time Analytics with Tenant Isolation & Cost Attribution

You're the Staff Engineer responsible for designing a real-time analytics platform that serves 10,000+ enterprise tenants. Each tenant ingests 10K-10M events/second with payloads ranging from 100 bytes to 50KB. Tenants must have strict performance isolation—noisy neighbors cannot impact others' query latency. The platform must support SQL-like queries with p99 latency <2s across petabyte-scale datasets while providing real-time cost attribution per tenant for showback/chargeback purposes. Additionally, tenants can define custom retention policies (7 days to 7 years) and the system must enforce GDPR/CCPA deletion requests within 24 hours across all storage tiers.

Key constraints: Cross-region deployment required (US-East, US-West, EU, APAC) with data residency compliance. Some tenants require strong consistency for specific event streams (financial audit trails), while others accept eventual consistency. The system must degrade gracefully under regional failures and support zero-downtime schema evolution as tenants frequently add new event fields. Your existing infrastructure runs on Kubernetes with Kafka, but you're free to propose alternatives where justified.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           INGESTION LAYER                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Region 1   │  │  Region 2   │  │  Region 3   │  │  Region 4   │         │
│  │ (US-East)   │  │  (US-West)  │  │    (EU)     │  │   (APAC)    │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                │                │
│         └────────────────┴───────┬────────┴────────────────┘                │
│                                  ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    STREAM PROCESSING (Flink/Kafka Streams)           │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │   │
│  │  │  Tenant ID   │  │   Schema     │  │   Cost Tag   │               │   │
│  │  │  Isolation   │  │  Registry    │  │   Injection  │               │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘               │   │
│  └──────────────────────────┬──────────────────────────────────────────┘   │
│                             │                                               │
│         ┌───────────────────┼───────────────────┐                          │
│         ▼                   ▼                   ▼                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐                 │
│  │  Hot Tier   │    │  Warm Tier  │    │    Cold Tier    │                 │
│  │ (ClickHouse)│    │  (Iceberg   │    │  (S3 + Athena)  │                 │
│  │  < 7 days   │    │  7-90 days  │    │   > 90 days     │                 │
│  └─────────────┘    └─────────────┘    └─────────────────┘                 │
│         │                   │                   │                          │
│         └───────────────────┼───────────────────┘                          │
│                             ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    QUERY LAYER (Query Router)                        │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │   │
│  │  │  Cost Meter  │  │  Cache Tier  │  │  Tenant QoS  │               │   │
│  │  │   Service    │  │  (Redis)     │  │  Enforcer    │               │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Components:**
- **Ingestion Routers**: Region-aware Kafka with tenant-aware partitioning
- **Stream Processors**: Flink for stateful processing with per-tenant resource quotas
- **Schema Registry**: Confluent or AWS Glue with versioned, per-tenant schemas
- **Storage Tiers**: Hot (real-time queries), Warm (batch analytics), Cold (compliance/audit)
- **Query Router**: Federates queries across tiers, enforces tenant isolation at compute layer
- **Cost Attribution**: Tagging every operation with tenant + resource dimensions

## Key Design Decisions

### 1. Tenant Isolation at the Kafka Partition Level
**Decision**: Use consistent hashing of `tenant_id` to dedicated partition ranges rather than separate topics.

**Trade-offs**:
- **Pros**: Better broker resource utilization (10K topics would exhaust metadata); supports true per-tenant throughput guarantees via partition-level quotas; simpler operational model.
- **Cons**: Rebalancing during tenant onboarding/offboarding is complex; partition count grows with tenant count (practical limit ~100K partitions per cluster); hot partitions possible if tenant ID distribution skews.

**Alternative**: Topic-per-tenant gives perfect isolation but creates Kafka metadata explosion. Consider hybrid: topic-per-tier (enterprise vs standard tenants) with partition hashing within.

### 2. Separation of Compute and Storage for Query Layer
**Decision**: Use ClickHouse for hot tier but store data in S3-backed Iceberg format for warm/cold tiers, with a unified query abstraction.

**Trade-offs**:
- **Pros**: Cost efficiency (S3 is 10x cheaper); independent scaling of compute and storage; support for time-travel queries via Iceberg snapshots; easier GDPR deletions (update metadata, don't rewrite parquet).
- **Cons**: Query latency for cold data suffers (S3 list operations); complex query planning (router must decide which tier to hit); data consistency challenges during tier transitions.

**Staff+ nuance**: How do you handle the "moving window" as data ages from hot→warm→cold? Streaming ingestion to both ClickHouse and Iceberg (lambda architecture) vs ClickHouse TTL export? Each has consistency and replay implications.

### 3. Cost Attribution via Resource Tagging vs Sampling
**Decision**: Tag every event and query with tenant dimensions at ingestion/router layer; meter actual CPU/memory/IO rather than estimating.

**Trade-offs**:
- **Pros**: Accurate showback/chargeback (customers care about precision); identifies true cost drivers (expensive regex queries vs simple aggregations); enables fair scheduling (weighted fair queuing based on spend).
- **Cons**: Tagging overhead (5-10% throughput reduction); storage cost for high-cardinality tags; complex to aggregate across distributed components; requires kernel-level eBPF or sidecar instrumentation for accuracy.

**Alternative**: Sampling is cheaper but loses accuracy for small tenants (variance matters at low volumes). Sampling + anomaly detection hybrid may be pragmatic.

## Deep Dive: Critical Components

### 1. Query Router with Tenant-Aware Scheduling

The query router is the critical path for isolation guarantees. It must:

- **Parse and rewrite queries**: Add implicit `WHERE tenant_id = X` clauses (prevents data leaks)
- **Cost estimation**: Use table statistics to predict query cost before execution
- **Admission control**: Maintain per-tenant token buckets for concurrent queries and CPU-seconds
- **Tier routing**: Route to ClickHouse for `event_time > now() - 7d`, Iceberg otherwise

**Critical implementation detail**: The router must implement **predictive backpressure**. If Tenant A submits 100 expensive full-table scans, naive fair queuing still hurts Tenant B's latency. Implement query cost classification (cheap <1s, medium 1-10s, expensive >10s) and limit expensive queries to N per tenant globally across regions. This requires distributed coordination (Redis/etcd) but prevents resource monopolization.

**Consistency challenge**: For strong consistency requirements (financial audit), queries must read from a specific Kafka offset that has been durably committed to ClickHouse. The router maintains a `(tenant_id, partition, offset)` watermark table—queries include `WHERE _offset <= watermark` to ensure repeatable reads. This trades latency (wait for watermark propagation) for consistency.

### 2. GDPR/CCPA Deletion Engine

Legal deletion within 24 hours across petabytes is non-trivial:

**Iceberg/S3 approach**: 
- Iceberg supports hidden deletes via equality deletes or positional deletes
- Mark rows as deleted in metadata layer; background compaction rewrites parquet files asynchronously
- Challenge: Compaction I/O costs; must throttle to prevent impacting queries

**ClickHouse approach**:
- Use mutations (`ALTER TABLE ... DELETE`)—but mutations are expensive and asynchronous
- Better: Partition by tenant_id + date, use TTL to drop entire parts
- Challenge: High cardinality of tenant_id (10K) × date (years) = too many parts; merge pressure

**Hybrid strategy**:
- Hot tier: Partition by `tenant_id % 100` + date; use `ALTER DELETE` within partition (limited scope)
- Warm/Cold: Iceberg hidden deletes with compaction schedule (off-peak hours)
- Deletion request workflow: Write to "deletion log" topic; Flink job propagates to all tiers; completion tracked in etcd; query router filters deleted rows until compaction completes

**Edge case**: What about derived tables/materialized views? Must propagate deletion to aggregates. Use "event_id" as immutable identifier; maintain reverse index from aggregate tables → source events. Deletion triggers aggregate recalculation.

### 3. Schema Evolution and Multi-Tenant Safety

Tenants frequently add fields (typical: 50 schema versions/tenant/year). The system must:

- **Enforce backward compatibility**: New fields must be optional with defaults; type changes prohibited without explicit migration
- **Handle divergent schemas**: Tenant A has `user_id` as string, Tenant B as int64—must store as variant or enforce registry-level validation
- **Query compatibility**: `SELECT new_field FROM events` should work for tenants with that field, return NULL for others

**Implementation**: 
- Avro/Protobuf schemas in registry with compatibility modes (BACKWARD, FORWARD, FULL)
- Storage layer uses columnar format with missing column handling (ClickHouse `DEFAULT`, Iceberg schema evolution)
- Query layer uses virtual views per tenant that project only columns in their schema

**Staff+ consideration**: What about cross-tenant queries (admin/analytics)? Create a unified view with `UNION ALL` of per-tenant views with column coercion. This is expensive—consider materialized "conformed schema" tables for admin queries, or restrict cross-tenant queries to common fields only.

## Follow-up Questions

1. **Consistency under failure**: A regional Kafka cluster fails during a tenant's peak ingestion. You have cross-region replication set up, but it introduces 100ms latency. For tenants requiring strong consistency, how do you handle the failover without duplicate events or data loss? What are the trade-offs between Kafka MirrorMaker 2, Confluent Replicator, and a custom consensus protocol at the application layer?

2. **Cost optimization edge case**: Tenant X discovers that grouping by a high-cardinality field (e.g., `user_id`) in queries is extremely expensive due to shuffle operations. They start issuing 1000x more queries but with `LIMIT 10`, hoping to get partial results faster. How does your cost attribution system account for query effort vs result size? How do you prevent this gaming of the system while maintaining fair pricing?

3. **Storage tiering strategy**: A tenant's data access pattern changes—they suddenly need to query 6-month-old data with the same 2s p99 latency requirement that applies to fresh data. The data is currently in S3/Iceberg (cold tier) where queries take 30s. You cannot migrate 50TB in real-time. What architectural options exist? Consider: predictive tiering based on access patterns, on-demand ClickHouse imports, or query result caching with materialized view incrementality. What are the cost and consistency implications of each?

---

*Generated: 2026-02-27 | Difficulty: Staff+ (20+ years experience) | Focus: Multi-tenant isolation, cost attribution, compliance, stream processing*
