---
title: Thanos Sharding for Long Term Retention Storage
type: proposal
menu: proposals
status: approved
owner: bwplotka
---

### Related Tickets

* https://github.com/thanos-io/thanos/issues/1034 (main)
* https://github.com/thanos-io/thanos/pull/1245 (attempt on compactor)
* https://github.com/thanos-io/thanos/pull/1059 (attempt on store)
* https://github.com/thanos-io/thanos/issues/1318 (subpaths, multitenancy)
* Store issues because of large bucket: 
  * https://github.com/thanos-io/thanos/issues/814
  * https://github.com/thanos-io/thanos/issues/1455

## Summary

This document describes the motivation and design of the sharding of Thanos components in terms of operations against object storage.
Additionally we touch smarter pre-filtering of shards on Querier.

## Motivation

Currently all components that read from object store assume that all the operations and functionality should be done based 
on **all** the available blocks available in the certain bucket's root directory. 

This is in most cases is totally fine, however with time and allowance of storing blocks from multiple `Sources` into the same bucket, 
the number of objects in a bucket can grow drastically.

This means that with time you might want to scale out certain components e.g:

* Compactor: Larger number of objects does not matter much, however compactor has to scale (CPU, network) with number of Sources pushing blocks to the object storage. 
If you have multiple sources handled by the same compactor, with slower network and CPU you might not compact/downsample quick enough to cope with incoming blocks.
    * This happens a lot if no compactor is deployed for longer periods and thus has to quickly catch up with large number of blocks (e.g couple of months).
* Store Gateway: Queries against store gateway which are touching large number of Sources might be expensive, so it has to scale up with number of Sources if we assume those queries.
    * Orthogonally we don't advertise any labels on Store Gateway's Info. This means that querier was not able to do any pre-filtering, so all store gateways in system are always touched for each query. 

### Use case

We can then define couple of use cases (some of them where already reported by users):

* Scaling out / Sharding Compactor.
* Scaling out / Sharding Store Gateways.
    * Advertising labels on Info labels.
* Filtering out portion of data: This is useful if you want to ignore suddenly some blocks in case of error/investigation/security.
* Different priority for different Sources.
    * Some Sources might be more important then others. This might mean different availability and performance SLOs. 
    Being able to split object storage operations across different components helps with that. NOTE: We mean here a per process priority (e.g one Store Gateway being more important then other). 

### Reminder: What is a Source 

`Source` is a any component that creates new metrics in a form of Thanos TSDB blocks uploaded to the object storage. We differentiate Sources by `external labels`. 
Having unique sources has several benefits:

 * Sources does not need to inject "global" source labels to all metrics (like `cluster, env, replica`). Those all the same for all metrics produced by source, we can assume that whole block has those.
 * We can track what blocks are "duplicates": e.g in HA groups, where 2 replicas of Prometheus-es are scraping the same targets.
 * We can track what source produced metrics in case of problems if any.

Example Sources are: Sidecar, Rule, Thanos Receive.

## Goals

Our goals for this design it to find and implement solution for:

* Sharding browsing metrics from the object storage:
  * e.g Selecting what blocks Store Gateway should expose.
* Pre-filtering which shards Querier should touch during query. 
  * This might mean advertise labels manipulation.
* Sharding compaction/downsampling of metrics in the object storage. 
  * NOTE: We need to be really careful to not have 2 compactors working on the same Source. This means careful upgrades/configuration changes. There must be documentation for that at least.
    
## No Goals

* Time partitioning
* "Merging" sources together virtually for downsampling/compactions across single Source that just changes external labels.
* Bloom filters for app metrics within blocks.
* Allow all permutations of object storage setups:
    * User uses multiple object storages for all sources?
    * User uses single object storage for all sources?
    * User uses any mix of object storages for sources. They put all in different subdirs/subpaths.
* Add coordination or reconciliation in case of multi Compactor run on the same "sources" or any form of Compactor HA (e.g active passive)
    * Requires separate design.

This design is orthogonal for multi-buckets/multi-prefix support. 

## Proposal

On each component that works on the object storage (e.g Store GW and Compactor), add `--selector.relabel-config` (and corresponding `--selector.relabel-config-file`) that will
be used to filter out what blocks should be selected for operations. Examples:

* We want to run Compactor only for blocks with `external_labels` being `cluster=A`. We will run second Compactor for blocks with `cluster=B` external labels.

Similar to [promtail](https://github.com/grafana/loki/blob/master/docs/promtail.md#scrape-configs) this config will follow native
[Prometheus relabel-config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config) syntax.

The relabel config will define filtering process done on **every** synchronization with object storage.

We will allow manipulating with several of inputs:

* `__block_id`
* External labels:
  * `<name>` 
* `__block_objstore_bucket_endpoint` 
* `__block_objstore_bucket_name`
* `__block_objstore_bucket_path`

Output:

* External labels to be advertised for this block.
* If output is empty, drop block. 

By default, on empty relabel-config, all external labels are assumed.
Intuitively blocks without any external labels will be ignored.

Example usages would be:

* Drop blocks which contains external labels cluster=A
```yaml
- action: drop
  regex: "A"
  source_labels:
  - cluster
```
* Keep only blocks which contains external labels cluster=A
```yaml
- action: keep
  regex: "A"
  source_labels:
  - cluster
```
* Drop cluster label from external labels for each blocks (if present).
```yaml
- action: labeldrop
  source_labels:
  - cluster
```
* Add `datacenter=ABC` external label to the result.
```yaml
- target_label: datacenter
  replacement: ABC
```

Result will depends on the component. For compactor result will be ignored. For store gateway, result will translate
into advertise labels exposed by Store Gateway on Info method.

### Work Plan

* Add/import relabel config into Thanos, add relevant logic
* Hook it for selecting blocks on Store Gateway
    * Advertise on resulted external labels.
* Hook it for selecting blocks on Compactor.
    * Add documentation about following concern: Care must be taken with changing selection for compactor to unsure only single compactor ever running over each Source's blocks.

### Future work

* Add coordination or reconciliation in case of multi Compactor run on the same "sources" or any form of Compactor HA (e.g active passive)
    * Requires separate design.