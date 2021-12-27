---
layout:              post
type:                classic
title:               The Decider System For Shard Allocation in Elasticsearch
subtitle:            >
    How does it works and what can we learn from it?

lang:                en
date:                2021-12-27 14:36:13 +0100
categories:          [elasticsearch]
tags:                [java, elasticsearch]
ads_tags:            [search]
comments:            true
excerpt:             >
    TODO

image:               /assets/bg-patrick-federi-LJEjI7LXABU-unsplash.jpg
cover:               /assets/bg-patrick-federi-LJEjI7LXABU-unsplash.jpg
article_header:
  type:              overlay
  theme:             dark
  background_color:  "#203028"
  background_image:
    gradient:        "linear-gradient(135deg, rgba(0, 0, 0, .6), rgba(0, 0, 0, .4))"
wechat:              false
---

## Introduction

Explain context here to attract people's attention... like:
- topic: what you want to talk about?
- audience: who are you targeting?
- motiviation: why is it interesting for them? Or why is it important to understand this topic?

After reading this article, you will understand:

* The responsibility of allocation service and allocation deciders.
* The structure of different deciders
* Some general concepts
* Some specific concepts to dig deeper
* How to go further from this article

Now, let's get started!

## Responsibility

Allocation deciders are part of the allocation service in Elasticsearch. This
service manages the node allocation of a cluster. For this reason the
`AllocationService` keeps `AllocationDeciders` to choose nodes for shard
allocation. This class also manages new nodes joining the cluster
and rerouting of shards. If none of the nodes accepts the allocation, the shard
will remain unassigned. Having unassigned shard leads to under-replicated
shards. It will probably result to yellow cluster and increase the risk of data loss.

_Which actions require decisions?_

We can find these information from the abstract class `AllocationDecider`.
An allocation decider can make decision for many actions: allocation,
rebalancing, keeping the shard in the current node (`canRemain`), and auto-expending the
shards of a given index. Each method returns a decision so that the allocation service how to
react. Each action respects the naming convention `can{Action}` or `should{Action}`.

```
➜  elasticsearch git:(v7.16.2 u=) ✗ grep "public Decision" server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/AllocationDecider.java | sort
    public Decision canAllocate(IndexMetadata indexMetadata, RoutingNode node, RoutingAllocation allocation) {
    public Decision canAllocate(ShardRouting shardRouting, RoutingAllocation allocation) {
    public Decision canAllocate(ShardRouting shardRouting, RoutingNode node, RoutingAllocation allocation) {
    public Decision canAllocateReplicaWhenThereIsRetentionLease(ShardRouting shardRouting, RoutingNode node, RoutingAllocation allocation) {
    public Decision canForceAllocateDuringReplace(ShardRouting shardRouting, RoutingNode node, RoutingAllocation allocation) {
    public Decision canForceAllocatePrimary(ShardRouting shardRouting, RoutingNode node, RoutingAllocation allocation) {
    public Decision canRebalance(RoutingAllocation allocation) {
    public Decision canRebalance(ShardRouting shardRouting, RoutingAllocation allocation) {
    public Decision canRemain(ShardRouting shardRouting, RoutingNode node, RoutingAllocation allocation) {
    public Decision shouldAutoExpandToNode(IndexMetadata indexMetadata, DiscoveryNode node, RoutingAllocation allocation) {
```

## Deciders Structure

In Elasticsearch 7.16, there are 19 deciders for the shard allocation. You can
find them as follows:

```
➜  elasticsearch git:(v7.16.2 u=) rg -l --sort-files "extends AllocationDecider" server/src/main | sed 's/.*\///g'
AllocationDeciders.java
AwarenessAllocationDecider.java
ClusterRebalanceAllocationDecider.java
ConcurrentRebalanceAllocationDecider.java
DiskThresholdDecider.java
EnableAllocationDecider.java
FilterAllocationDecider.java
MaxRetryAllocationDecider.java
NodeReplacementAllocationDecider.java
NodeShutdownAllocationDecider.java
NodeVersionAllocationDecider.java
RebalanceOnlyWhenActiveAllocationDecider.java
ReplicaAfterPrimaryActiveAllocationDecider.java
ResizeAllocationDecider.java
RestoreInProgressAllocationDecider.java
SameShardAllocationDecider.java
ShardsLimitAllocationDecider.java
SnapshotInProgressAllocationDecider.java
ThrottlingAllocationDecider.java
```

Among these deciders, there is a root decider, the `AllocationDeciders`, which
asks all other deciders to decide for the shard allocation. As you can see from
the file name, they are for awareness key-value pairs (such as availability
zone), cluster rebalancing, concurrent relabalancing, disk threshold, and much
more.

## Making Decisions

* Structure of the decisions
* Processing mode (fetch-then-process)
* When are the decisions made?

## Lifecycle

_When are deciders created? I.e. where are deciders positioned in the lifecycle of
an Elasticsearch cluster?_

When starting a new node, the class `Bootstrap` is called. Before starting the
node, it creates a new `Node` instance, which creates and addes a list of modules. One of these modules is
called `ClusterModule` and it contains the deciders. So the whole logic happens
at the early stage of the lifecycle, more precisely, the creation happens before the
startup of a node.

```yml
- boostrap
  - construct node
    - create modules
      - create master module
        - create deciders
        - create root decider (AllocationDeciders)
        - create allocation service
        - ...
    - add modules
  - start node
```

## Testing

* How are they tested?

## Going Further

How to go further from here?

TODO:

* Who asks the node the make decisions? Does every node participate to the
  decision or only one node handles everything for the cluster?

## Conclusion

What did we talk in this article? Take notes from introduction again.
Interested to know more? You can subscribe to [the feed of my blog](/feed.xml), follow me
on [Twitter](https://twitter.com/mincong_h) or
[GitHub](https://github.com/mincong-h/). Hope you enjoy this article, see you the next time!

## References

- ["Elasticsearch Shards"](https://opster.com/guides/elasticsearch/glossary/elasticsearch-shards/),
  Opster, 2021.
