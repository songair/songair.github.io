---
layout:              post
type:                classic
title:               The Decider System For Shard Allocation in Elasticsearch
subtitle:            >
    How does it work and what can we learn from it?

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

Shard allocation plays an important role in the data distribution of an
Elasticsearch cluster. Having all the shards allocated ensures the good health
of you cluster and avoids potential data loss. That's why it's important to
avoid unassigned shards and avoid yellow cluster. But as a software engineer or
as a database administrator (DBA), do you know how the decision is made
internally? How does Elasticsearch know which node should be used? In this
article, we are going to study this part by exploring the decider system.

After reading this article, you will understand:

* The responsibility of allocation service and allocation deciders.
* The structure of these 19 deciders.
* How do they make decisions?
* The deciders' position in the lifecycle of an ES node.
* How to test them?
* How to go further from this article?

And hopefully, this article helps you better understand the internal mechanism
or inspire you to make your decision making system based on similar
architecture. Note that this article is written with Elasticsearch 7.15.2
(latest), which may be different from what you are using. Now, let's get started!

## Responsibility

Allocation deciders are part of the allocation service in Elasticsearch. This
service manages the node allocation of a cluster. For this reason, the
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
react. Each action respects the naming convention `can{Action}` or
`should{Action}`. Here is a more detailed version:

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
contains the references of all other deciders. When making a decision, it will
ask those deciders to decide for the shard allocation, and then make a global
decision based on the results. We will discuss more details in the following
section. As for other deciders, you can see their purposes from
their filenames, such as for awareness key-value pairs (e.g. availability zone),
cluster rebalancing, concurrent relabalancing, disk threshold, and much more.

## Making Decisions

Now let's take a look at the `AllocationDeciders` to see how it makes a
decision. When starting the decision, it accepts information about the shard
routing, node routing and the current allocation. Then, it either ignores the
shard or delegate the decision making to its child deciders. All child
deciders make decision in a synchronous way, probably because they don't require
additional information (such as HTTP request) from an external service. Once the
decision is made by the child decision, the result is processed with the fail-fast
strategy. That is, whenever one child decision is negative (NO), we consider the
whole decision as NO without asking the remaining deciders. It's a short
circuit. We do that unless we are debugging the decision, in which case, we
gather all the decisions, including the NO decisions, before returning the final
result.

```java
    @Override
    public Decision canAllocate(ShardRouting shardRouting, RoutingNode node, RoutingAllocation allocation) {
        if (allocation.shouldIgnoreShardForNode(shardRouting.shardId(), node.nodeId())) {
            return Decision.NO;
        }
        Decision.Multi ret = new Decision.Multi();
        for (AllocationDecider allocationDecider : allocations) {
            Decision decision = allocationDecider.canAllocate(shardRouting, node, allocation);
            // short track if a NO is returned.
            if (decision.type() == Decision.Type.NO) {
                if (logger.isTraceEnabled()) {
                    ...
                }
                // short circuit only if debugging is not enabled
                if (allocation.debugDecision() == false) {
                    return Decision.NO;
                } else {
                    ret.add(decision);
                }
            } else {
                addDecision(ret, decision, allocation);
            }
        }
        return ret;
    }
```

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
