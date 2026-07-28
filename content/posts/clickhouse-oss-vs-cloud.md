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

## Keeper becomes part of the scaling boundary

`ReplicatedMergeTree` uses ClickHouse Keeper to coordinate replication metadata. Keeper does not store the table data itself, but it sits on the control path for replicated tables, parts, mutations, and other coordination work.

Once replication is introduced, the ClickHouse cluster is no longer the only system the team operates. Keeper needs its own odd-sized quorum, durable storage, backups, monitoring, upgrades, latency budgets, and failure procedures.

As the number of replicated tables, parts, replicas, and metadata operations grows, Keeper receives more coordination traffic. A workload that creates many small parts or performs frequent schema and mutation operations can make that pressure visible quickly. Scaling the ClickHouse data nodes does not automatically scale away a Keeper bottleneck.

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

## A simple TCO comparison

A useful comparison starts with two different equations.

For self-managed ClickHouse:

```text
OSS TCO =
  (shards × replicas × data-node cost)
  + Keeper
  + backups and network
  + idle failover/headroom
  + engineering and on-call time
```

For ClickHouse Cloud:

```text
Cloud TCO =
  metered compute
  + object storage
  + data transfer
  + managed-service premium
  + the smaller amount of database work the team still owns
```

The first formula is dominated by the provisioned topology. The second is dominated by compute actually kept running and data retained. ClickHouse Cloud can scale compute separately from storage and can scale idle services to zero, according to its [pricing model](https://clickhouse.com/pricing). That matters when traffic is bursty or the cluster has to carry a lot of spare capacity.

There is a useful public price anchor, although it is not a universal quote. In an [official comparison published with November 2025 AWS us-east prices](https://clickhouse.com/blog/how-cloud-data-warehouses-bill-you), ClickHouse listed one compute unit as **2 vCPU and 8 GiB of memory**, at **$0.22/hour for Basic, $0.30/hour for Scale, and $0.39/hour for Enterprise**. The same comparison used **$25.30 per TB-month** for Cloud storage. Prices vary by region and tier, so the current [pricing page](https://clickhouse.com/pricing) should be used for an actual decision.

### An illustrative AWS deployment

To put numbers behind the comparison, assume a self-managed deployment in AWS us-east-1 with:

- One `i4i.4xlarge` per shard replica: 16 vCPU, 128 GiB RAM, and 3.75 TB local NVMe, assumed at **$1.373/hour**, or roughly **$1,002/month**. The hardware specification comes from the [AWS storage-optimized instance documentation](https://docs.aws.amazon.com/ec2/latest/instancetypes/so.html).
- Two replicas per shard across availability zones.
- Three small Keeper nodes at roughly **$74/month each**.
- One S3 backup copy of the logical data, assumed at **$23/TB-month**.
- 730 hours per month, On-Demand pricing, no Savings Plan, taxes, support, cross-AZ traffic, monitoring, or snapshot request charges.

These are deliberately round assumptions, not a quote. AWS notes that On-Demand instances are billed by actual running time, while committed-use discounts can materially change the result; its current rates should be checked on the [EC2 pricing page](https://aws.amazon.com/ec2/pricing/).

The infrastructure bill then looks like this:

| OSS topology | Data nodes | Keeper | Backup | Approx. infra/month |
| --- | ---: | ---: | ---: | ---: |
| `1×2` | $2,004 | $222 | $69 | **$2,295** |
| `2×2` | $4,008 | $222 | $138 | **$4,368** |
| `4×2` | $8,016 | $222 | $276 | **$8,514** |
| `8×2` | $16,032 | $222 | $552 | **$16,806** |

Now create a rough Cloud comparison. Matching the 128 GiB memory of one logical shard requires 16 ClickHouse Cloud compute units under the public definition above. This is a **memory-capacity comparison, not a performance benchmark**: the CPU ratio, storage path, caching, and query behavior differ.

At the Scale list price, 16 compute units running continuously cost:

```text
16 CU × $0.30 × 730 hours = $3,504/month
```

Adding 3 TB of Cloud storage gives roughly **$3,580 per logical shard per month**. With continuous compute, the illustrative comparison becomes:

| Logical capacity | OSS infra | Cloud, 100% compute uptime |
| --- | ---: | ---: |
| 1 shard | $2,295 | $3,580 |
| 2 shards | $4,368 | $7,160 |
| 4 shards | $8,514 | $14,320 |
| 8 shards | $16,806 | $28,640 |

On raw infrastructure alone, steady 24/7 OSS wins this example. That is the Cloud premium in plain numbers.

But Cloud compute is elastic. If the same workload averages 60% of its peak compute allocation, the eight-shard Cloud estimate falls from about **$28,640 to $17,425**. It is then close to the **$16,806** OSS infrastructure bill before a single hour of ClickHouse operations is counted.

That last part changes the answer. Using a hypothetical fully loaded engineering cost of **$150/hour**, the continuous-compute Cloud premium at `1×2` is equivalent to about **nine operator-hours per month**. At `8×2`, it is about **79 hours per month**. If the self-managed cluster consumes more time than that in capacity planning, upgrades, rebalancing, incidents, and restore testing, Cloud has the lower TCO even though its raw compute rate is higher.

The corresponding OSS comparison should therefore use the **fully loaded** monthly cost of the production topology, not the price of one VM:

```text
break-even Cloud bill =
  data nodes + replicated disks + Keeper + backups
  + unused headroom + monthly operator cost
```

If one engineer spends 20 hours a month on upgrades, capacity planning, replication lag, backup tests, and incidents, that time belongs in the calculation. So does the cost of a resharding project, spread across the period it benefits.

## Where the crossover usually appears

There is no honest answer such as “Cloud wins above 20 TB.” Data volume alone does not determine the compute load, query concurrency, compression ratio, or operational burden. The crossover is better described by workload shape.

| Situation | Usually favors |
| --- | --- |
| Small, steady, 24/7 workload on one well-utilized server | OSS |
| Existing platform team already operates ClickHouse well | OSS |
| Cheap local NVMe and predictable long-running utilization | OSS |
| Bursty traffic with long idle periods | Cloud |
| Storage grows much faster than query compute | Cloud |
| Frequent scaling, resharding, or topology changes | Cloud |
| Small team where database operations interrupt product work | Cloud |
| Multiple isolated read workloads over the same data | Cloud |

This is the part that a raw instance-price comparison misses. At small scale, the Cloud premium is visible and a simple OSS deployment can be dramatically cheaper. As the OSS topology grows from `1×2` to `4×2` or `8×2`, replicated disks, failover headroom, Keeper, and operational work grow with it. Cloud becomes economical when its premium is lower than those avoided costs:

```text
Cloud premium
<
replication overhead + idle capacity + Keeper
+ rebalancing work + routine operations + incident risk
```

That crossover often arrives earlier for a fast-growing team than for a mature infrastructure organization. It also arrives earlier for spiky workloads than for a cluster that runs near full utilization all day.

But “Cloud always wins at large scale” is too strong. A very large, predictable workload with high utilization and a capable platform team can keep OSS infrastructure cheaper, even after replication. At that point the decision is less about scale itself and more about whether Cloud's elasticity and reduced operational load are worth the premium.

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

This does not make distributed systems disappear. New compute still needs CPU, memory, local cache warm-up, and coordination. Object storage introduces its own latency and cost model. ClickHouse Cloud also charges a managed-service premium and gives the operator less control over versions, configuration, and infrastructure.

But it removes a particularly expensive coupling: in the shared-nothing model, scaling compute, moving data, and changing topology are often the same operation. With shared storage, they can be separate operations.

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
