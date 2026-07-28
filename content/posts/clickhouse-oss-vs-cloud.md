---
title: "The hidden cost of running ClickHouse OSS at scale"
date: 2026-07-25
tags: [clickhouse, analytics]
summary: "ClickHouse OSS is easy to start and fast on one node. The hidden cost appears when growth turns one node into shards, replicas, Keeper, and a distributed system your team must operate."
---

ClickHouse OSS is remarkably easy to start. Put it on a large machine, point data at it, and it will take you surprisingly far.

The difficult part begins when one machine is no longer enough.

I have spent a lot of time operating self-managed ClickHouse clusters, and the costs that matter most rarely appear in the first infrastructure estimate. They emerge later: when a shard fills up, when another replica is needed, when replication falls behind, when Keeper becomes the coordination hot spot, or when a topology change turns into a migration project.

These are not bugs in ClickHouse. They are consequences of its shared-nothing architecture.

## The shared-nothing bargain

In a typical self-managed ClickHouse cluster, each node owns local data and local compute. To store more data or use more CPU, you divide the dataset into **shards**. To survive failures or add read capacity, you create **replicas** of those shards.

The model is simple:

- Shards split the data.
- Replicas copy the data.
- `Distributed` tables route work across shards.
- `ReplicatedMergeTree` and ClickHouse Keeper coordinate replicas.

![ClickHouse self-managed architecture: shards, replicas, Keeper, and storage are deployed and operated together](https://clickhouse.com/_next/static/media/architecture-oss.2uhq7kjponoex.png)

*ClickHouse's self-managed reference architecture. Source: [ClickHouse Cloud architecture comparison](https://clickhouse.com/cloud).*

This design performs well because computation happens close to local data. It also means that storage layout, compute capacity, and cluster topology are coupled. Adding a server is not the same as adding usable capacity.

That coupling is where the hidden cost starts.

## Sharding makes scaling a data-placement problem

Suppose a cluster begins with one shard and two replicas. When the shard approaches its storage or compute limit, adding another shard does not automatically redistribute existing data.

There are two common reasons to add shards:

- **Storage capacity:** local disks are filling up, so the dataset has to be partitioned across more machines.
- **Ingestion capacity:** a single shard cannot parse, sort, compress, and merge incoming data fast enough, so writes have to be spread across more CPUs.

The second limit can arrive before the first. ClickHouse ingestion is often CPU-bound, especially with high row rates, expensive materialized views, complex codecs, or sustained background merges. Ingestion and queries then compete for the same CPU, memory bandwidth, and disk I/O. A node can have plenty of free storage while query latency deteriorates because writes and merges consume its compute budget.

Adding shards increases aggregate ingestion capacity by distributing that work. It also changes the data layout, however, so a compute bottleneck becomes a topology change rather than a simple CPU allocation. Adding replicas does not fully solve it either: replicas can increase read capacity, but each replica still performs replication work and background merges.

You now have to make several decisions:

- What is the new sharding key?
- Does historical data need to move?
- How will writes transition to the new topology?
- Can queries tolerate data living under two layouts during the migration?
- How do you validate that nothing was duplicated or missed?

Even when old data stays where it is and only new writes use the new shard, the cluster becomes less balanced over time. If data must be redistributed, scaling becomes an I/O-heavy migration with operational risk.

Query behavior changes too. A distributed query fans out to the relevant shards and waits for their results. One overloaded or unhealthy shard can determine tail latency for the entire query. The more shards a query touches, the more network calls and failure surfaces it acquires.

The hardware is only the visible cost. The hidden cost is designing, executing, observing, and sometimes rolling back the topology change.

## Replicas multiply more than storage

Replicas are necessary for high availability and can increase read throughput, but a replica is not free compute attached to existing storage. It is another full copy of the shard.

Adding a replica means:

- Copying or fetching all of the shard's data.
- Paying for another full set of disks.
- Running background merges on another server.
- Warming another local cache.
- Routing traffic so the extra read capacity is actually used.
- Monitoring replication queues and lag.
- Planning for the network and I/O load of recovery after a failure.

A cluster with two shards and two replicas already has four data-bearing nodes. Moving to four shards with two replicas doubles that footprint to eight. If storage and compute requirements grow at different rates, the topology cannot express that cleanly: scaling one often means paying for more of the other.

Replication also changes failure recovery. A replacement node may be easy to provision, but it is not useful until it has recovered enough data and cache state to serve the workload safely. At large data volumes, that recovery path deserves the same capacity planning as normal traffic.

It is worth being precise about the shape of this cost. `shards × replicas` is **multiplicative**, not mathematically exponential. If the replica count stays at two, doubling the shards doubles the data-node fleet:

| Topology | Data-bearing nodes |
| --- | ---: |
| 1 shard × 2 replicas | 2 |
| 2 shards × 2 replicas | 4 |
| 4 shards × 2 replicas | 8 |
| 8 shards × 2 replicas | 16 |

What makes the bill feel worse than linear is the way capacity arrives in steps. A new shard normally brings its replicas, disks, cache, merge capacity, and failure-domain placement with it. The cluster is sized for the hottest shard and for failover, not for the average CPU graph. That leaves paid capacity idle between growth events.

The other dimension is read capacity. Replicas can serve independent queries when traffic is balanced across them, so a shard that is saturated by concurrent reads may need another replica even when it has enough storage and ingestion capacity. ClickHouse's own [concurrency sizing guidance](https://clickhouse.com/resources/engineering/high-concurrency-sizing-user-analytics) makes an important distinction: replicas add read throughput, while parallel replicas allow a suitable single query to use multiple replicas. In the normal distributed path, simply adding a replica does not automatically make one query faster.

Technically, a team can add a third replica only to the hot shard. Operationally, that creates an asymmetric cluster: shards now have different read capacity, failure tolerance, cache state, and routing behavior. Many teams keep the topology uniform instead. Moving from two replicas to three then adds one complete copy of **every** shard:

| Topology change | Nodes before | Nodes after | Increase |
| --- | ---: | ---: | ---: |
| 4 shards: `4×2 → 4×3` | 8 | 12 | +4 nodes / +50% |
| 8 shards: `8×2 → 8×3` | 16 | 24 | +8 nodes / +50% |
| 20 shards: `20×2 → 20×3` | 40 | 60 | +20 nodes / +50% |
| 8 shards: `8×2 → 8×4` | 16 | 32 | +16 nodes / +100% |

That is the expensive step hidden behind “add read capacity.” A local problem on one shard can turn into `M` new servers because the production topology is kept symmetric. Each server needs the shard's full data, performs replication and background merges, warms its own cache, and consumes recovery bandwidth. Going from two replicas to three is not a small adjustment; it increases the entire data-node fleet by 50%.

## Keeper becomes part of the scaling boundary

`ReplicatedMergeTree` uses ClickHouse Keeper to coordinate replication metadata. Keeper does not store the table data itself, but it sits on the control path for replicated tables, parts, mutations, and other coordination work.

Once replication is introduced, the ClickHouse cluster is no longer the only system the team operates. Keeper needs its own odd-sized quorum, durable storage, backups, monitoring, upgrades, latency budgets, and failure procedures.

Adding replicas increases this burden as well as the data-node bill. Every new replica registers itself, maintains replication state and queues, watches metadata, reports part state, and participates in mutations and recovery. A change from `8×2` to `8×3` adds eight full ClickHouse servers, but those servers all attach to the same Keeper control plane.

The important limit is Keeper's write path. Keeper uses [Raft](https://clickhouse.com/clickhouse/keeper): clients can connect to different Keeper nodes, but writes are ordered through the current leader, appended to its log, and replicated to a quorum before they are committed. Followers improve availability; they do not create independent write leaders. Keeper write throughput is therefore bounded by the leader's CPU and durable-log latency, plus the network and disk latency needed to reach a majority.

This makes coordination writes a finite cluster-wide budget. The ClickHouse [replication documentation](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replication) says that each inserted block creates approximately ten Keeper entries through several transactions. Small insert batches create more blocks, and therefore far more coordination work for the same number of rows. Frequent part creation, schema changes, mutations, replica churn, and recovery add traffic on top.

The rough shape is:

```text
Keeper pressure =
  inserted blocks and part metadata
  + replicated tables × replicas
  + mutations and DDL
  + replica recovery and churn
```

Adding more Keeper quorum members does not scale that write path horizontally because there is still one leader. Scaling the ClickHouse data nodes does not remove the bottleneck either; it can send even more sessions, watches, and metadata operations toward the same leader.

ClickHouse supports `auxiliary_zookeepers`, which allows different tables to place their replication metadata in different Keeper or ZooKeeper clusters. This can partition the write load. It also changes one control plane into several:

- Another odd-sized Keeper quorum to provision and place across failure domains.
- Another set of disks, snapshots, alerts, certificates, and upgrade procedures.
- Configuration and credentials distributed to every ClickHouse node that uses it.
- A permanent table-to-Keeper mapping that operators must understand during incidents.
- More recovery and migration procedures to test.

That is a valid escape hatch for a very large deployment, but it is not free scale-out. Once Keeper has to be sharded by table, coordination capacity has become an architecture of its own, and the operational cost rises close to linearly with the number of Keeper clusters.

In practice, this creates another class of incidents to understand:

- Is replication slow because a replica is overloaded?
- Is the network dropping coordination requests?
- Is Keeper latency increasing?
- Are there too many parts or queued operations?
- Will restarting a node reduce pressure or trigger an expensive recovery?

Keeper is a solid coordination system. The hidden cost is that coordination itself becomes production infrastructure owned by your team.

## The operational bill is larger than the server bill

The raw infrastructure comparison between ClickHouse OSS and ClickHouse Cloud is tempting: add up instances and disks, compare the number with the managed-service price, and declare self-hosting cheaper.

That comparison leaves out the work required to keep a shared-nothing cluster healthy:

- Capacity planning for each shard.
- Rebalancing and resharding.
- Replica placement across failure domains.
- Keeper operation and recovery.
- Backups and restore testing.
- Rolling upgrades across compatible versions.
- Configuration consistency across nodes.
- Monitoring merges, parts, queues, disks, and query fan-out.
- On-call time when any of these systems interact badly.

For a large, steady workload with an experienced platform team, self-managed ClickHouse can still be the right economic choice. But the honest cost is not just compute plus storage. It includes the engineering time and operational risk created by the topology.

## What shared storage changes

ClickHouse Cloud uses `SharedMergeTree`: table data lives in shared object storage, while compute replicas process that data. The durable copy of the dataset is no longer tied to the lifecycle of an individual compute node. ClickHouse describes this as the progression from shared-nothing servers with local state to [stateless compute over shared data](https://clickhouse.com/blog/clickhouse-cloud-stateless-compute).

![ClickHouse Cloud architecture: an elastic compute layer separated from shared object storage](https://clickhouse.com/_next/static/media/architecture-cloud.1a01-xtsxepqq.png)

*ClickHouse Cloud separates the compute fleet from shared object storage and adds a managed control plane. Source: [ClickHouse Cloud architecture comparison](https://clickhouse.com/cloud).*

That changes the scaling problem:

- Adding compute does not require creating another full durable copy of the data.
- Replacing a compute node does not require rebuilding the dataset from another replica.
- Storage and compute can grow more independently.
- Multiple compute services can use the same underlying data.
- Horizontal scaling does not require manually resharding the table first.

This does not make distributed systems disappear. New compute still needs CPU, memory, cache warm-up, and coordination. Object storage introduces its own latency and cost model. ClickHouse Cloud also charges a managed-service premium and gives the operator less control over versions, configuration, and infrastructure.

But it removes the coupling that matters for the cost model below: in shared-nothing OSS, storage, ingestion capacity, query capacity, and replication are expressed through the same fleet. In Cloud, they can be priced and scaled more independently.

## A capacity-based TCO comparison

A useful TCO model has to size both systems for work, not memory. The relevant questions are:

- How much data must remain queryable?
- How many rows per second must become query-ready?
- How much analytical query work must be served?
- How much headroom is required while ingest and queries run together?

### Use ClickBench as a calibration point

[ClickBench](https://github.com/ClickHouse/ClickBench) loads 100 million web-analytics rows and runs 43 analytical queries three times. I used its latest published AWS results as calibration points, then derived two rates:

```text
ingest rows/second = 100,000,000 / load time

hot ClickBench-equivalent queries/hour =
  43 × 3,600 / sum(best of run 2 and run 3)
```

“ClickBench-equivalent queries” are units of analytical work, not application QPS. One production query that scans one tenth as much data might count as roughly 0.1 unit; a larger join or scan could count as several. ClickBench is sequential, not a concurrency benchmark.

The derived results are:

| Deployment | Ingest | Hot 43-query cycle | Query work/hour |
| --- | ---: | ---: | ---: |
| OSS `c6a.xlarge`, 1 node | 0.29M rows/s | 134.414s | 1,152 |
| OSS `c6a.4xlarge`, 1 node | 0.34M rows/s | 24.275s | 6,377 |
| Cloud `2×12 GiB` | 0.55M rows/s | 134.928s | 1,147 |
| Cloud `2×32 GiB` | 1.38M rows/s | 45.611s | 3,394 |
| Cloud `2×120 GiB` | 4.78M rows/s | 13.450s | 11,509 |
| Cloud `2×236 GiB` | 8.55M rows/s | 8.859s | 17,474 |

Sources: [OSS result](https://github.com/ClickHouse/ClickBench/blob/main/clickhouse/results/20260722/c6a.4xlarge.json) and [Cloud results](https://github.com/ClickHouse/ClickBench/tree/main/clickhouse-cloud/results/20260706).

The curves are different. On the published OSS setup, moving from `c6a.xlarge` to `c6a.4xlarge` gives four times the vCPU, and the hot query-work rate improves about 5.5×. Ingestion improves only about 19%. On that machine, the load path is already constrained by more than CPU: storage, fsync, sorting, and part creation matter.

Cloud's published service sizes show a different curve because compute, shared storage, cache, and parallel replicas are different. That is exactly why “match the RAM and compare the invoice” is the wrong model.

There are important caveats. The OSS load reads local Parquet files; the Cloud load reads a remote URL and enables parallel replicas. The OSS result is not replicated, so it does not include replica fetches and repeated merge work. Ingest and queries are measured separately, not simultaneously. These numbers are calibration points for an example, not a substitute for replaying a production workload.

### Turn the benchmark into a capacity envelope

For OSS, let `M` be shards and `R` be replicas. Using the `c6a.4xlarge` result, a simple planning constraint is:

```text
OSS load =
  ingest rate / (M × 341k rows/s)
  + query work / (R × 6,377 per hour)
  ≤ 75%
```

Shards divide ingestion across data partitions. Replicas let independent queries use different copies of those shards. The remaining 25% is headroom for merges, replication, failover, and imperfect balance.

Storage adds another constraint. The ClickBench machine uses a 500 GB volume. If only 70% is treated as safely usable, then:

```text
M ≥ compressed data / 350 GB
```

For Cloud, choose the smallest published two-replica service where:

```text
Cloud load =
  ingest rate / measured service ingest capacity
  + query work / measured service query capacity
  ≤ 75%
```

This is still simplified, but it forces ingestion and query capacity into the same decision instead of assuming every node is interchangeable.

### Price the capacity

For an illustrative AWS us-east-1 comparison, I used:

- `c6a.2xlarge`: about **$223/month** On-Demand compute.
- `c6a.4xlarge`: about **$447/month** On-Demand compute.
- One 500 GB `gp3` volume per data node: **$40/month**.
- A three-node Keeper quorum: **$222/month** total.
- One S3 backup copy: **$23/TB-month**.
- ClickHouse Cloud Scale: **$0.30 per compute unit/hour**, with 1 CU defined as 2 vCPU and 8 GiB.
- ClickHouse Cloud storage: **$25.30/TB-month**.
- 730 running hours per month.
- OSS operator time: **$150/hour** fully loaded.

The AWS prices are round assumptions based on [EC2 On-Demand](https://aws.amazon.com/ec2/pricing/) and [gp3 pricing](https://aws.amazon.com/ebs/pricing/). The Cloud rate is the public price used in ClickHouse's [cost methodology](https://clickhouse.com/blog/how-cloud-data-warehouses-bill-you). No volume discounts, Savings Plans, support, taxes, cross-AZ traffic, or internet egress are included.

### Four scaling stages

The workload grows along all three axes:

| Stage | Compressed data | Ingest | Query work/hour |
| --- | ---: | ---: | ---: |
| Small | 0.2 TB | 50k rows/s | 500 |
| Growth | 0.8 TB | 300k rows/s | 1,500 |
| Scale | 1.5 TB | 800k rows/s | 4,000 |
| Large | 3.0 TB | 1.5M rows/s | 10,000 |

Applying the 75% capacity envelope produces:

| Stage | OSS deployment | OSS infra | Ops h/month | OSS TCO | Cloud deployment | Cloud cost |
| --- | --- | ---: | ---: | ---: | --- | ---: |
| Small | `1×2`, `c6a.2xlarge` | $0.75k | 6 | **$1.65k** | `2×12 GiB` | **$0.66k** |
| Growth | `3×2`, `c6a.2xlarge` | $1.82k | 14 | **$3.92k** | `2×32 GiB` | **$1.77k** |
| Scale | `6×2`, `c6a.4xlarge` | $6.10k | 32 | **$10.90k** | `2×120 GiB` | **$6.61k** |
| Large | `13×4`, `c6a.4xlarge` | $25.62k | 80 | **$37.62k** | `2×236 GiB` | **$13.00k** |

The operational hours are estimates, not ClickBench measurements:

- **6 hours** for a small HA cluster: patching, backup checks, alerts, and occasional replication work.
- **14 hours** at `3×2`: more capacity review, rolling work, and replica investigation.
- **32 hours** at `6×2`: regular topology work plus incidents and restore tests amortized monthly.
- **80 hours** at `13×4`: a material on-call and platform workload, including Keeper and rebalancing projects.

Schema design, query tuning, and data modeling are excluded because both options still require them. The hours above cover work the managed control plane substantially reduces.

### Where the crossover happens

At the **Scale** stage, self-managed infrastructure is still slightly cheaper: **$6.10k versus $6.61k**. Once 32 operator-hours are included, OSS reaches **$10.90k** and Cloud has the lower TCO.

At the **Large** stage, the workload needs both more ingestion partitions and more read concurrency. The illustrative OSS choice becomes `13×4`: 52 data-bearing nodes. Cloud scales compute in front of one shared dataset. Its **$13.00k** managed bill is already below the **$25.62k** OSS infrastructure bill before operational time.

That is the real crossover. It is not “when `M × N` equals 40.” It happens when the capacity constraints force both `M` and `R` upward:

```text
storage or ingestion pressure → more shards
read-concurrency pressure      → more replicas
both at once                   → M × R data nodes
```

The result is sensitive to the assumptions. Savings Plans, local NVMe, better batching, and a team that already operates Keeper can move the OSS line down. Bursty traffic, a third or fourth replica, auxiliary Keeper clusters, or frequent resharding move it up. The useful output is not the exact dollar; it is a model that can be replaced with measured ingest, query-work, utilization, and operator hours from the real cluster.

## The real trade-off

The difference between ClickHouse OSS and ClickHouse Cloud is not simply free software versus a cloud bill.

It is a choice about where complexity lives.

With self-managed ClickHouse, the software is open and the infrastructure can be cheaper, but the user owns sharding, replication, coordination, recovery, and the people-hours behind them. With ClickHouse Cloud, more of that complexity is absorbed by the platform, and the premium appears directly on the invoice.

My practical rule is:

- Choose OSS when the workload is stable, the hardware will stay busy, and the team already has the operational capability.
- Choose Cloud when growth is uncertain, utilization is uneven, or avoiding topology and on-call work is more valuable than minimizing the raw infrastructure bill.

The decision should be revisited as the topology changes. A `1×2` cluster and an `8×2` cluster are not the same economic product, even if both are called “self-managed ClickHouse.”

After operating the OSS architecture, the question I keep coming back to is not whether shared storage is useful. It clearly is.

The more interesting question is whether we can get its decoupled scaling model without giving up the control and economics that make open-source ClickHouse attractive.

That is the architecture I want to explore next.
