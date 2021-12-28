---
layout:              post
type:                classic
title:               The Decider System For Shard Allocation in Elasticsearch
subtitle:            >
    How does it work and what can we learn from it?

lang:                en
date:                2021-12-27 14:36:13 +0100
categories:          [elasticsearch]
tags:                [java, elasticsearch, system-design]
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

There are two types of decision in the decider system: single decision and
multi-decision. A single decision represents a decision made on one given
dimension, e.g. awareness or disk threshold. While a multi-decision represents a decision
container which contains a list of child decisions. Let's take a closer look into
both of them.

### Single Decision

`DiskThresholdDecider` is a good example of making a single decision. Inside
the method for shard allocation
(`canAllocate(...)`), first of all, it retrieves the settings from the class member
`diskThresholdSettings` for the low and high thresholds, then it retrieves the disk
usage from the given information (node, routing allocation, usages), and finally
compare both of them to obtain the decision.
Depending on the situation, we will either get a YES or NO decision.

```java
    @Override
    public Decision canAllocate(ShardRouting shardRouting, RoutingNode node, RoutingAllocation allocation) {
        ...

        // retrieve settings
        final double usedDiskThresholdLow = 100.0 - diskThresholdSettings.getFreeDiskThresholdLow();
        final double usedDiskThresholdHigh = 100.0 - diskThresholdSettings.getFreeDiskThresholdHigh();

        // checks for exact byte comparisons
        if (freeBytes < diskThresholdSettings.getFreeBytesThresholdLow().getBytes()) {
           ...
        }

        // checks for percentage comparisons
        if (freeDiskPercentage < diskThresholdSettings.getFreeDiskThresholdLow()) {
           ...
        }

        ...
    }
```

Note that YES and NO are not the only types for the cluster. It also exists a
THROTTLE type, which allows users to controle the shard allocation based on the number
of actions performed concurrently. We can consider it as a "temporary NO", as
the allocation is not allowed right now, but may be YES in the future.

### Multi-Decision

Now let's take a look at the `AllocationDeciders` to see how it makes a
multi-decision. When starting the decision, it accepts information about the shard
routing, node routing and the current allocation. Then, it either ignores the
shard or delegate the decision making to its child deciders. All child
deciders make decision in a synchronous way, probably because they don't require
additional information (such as HTTP request) from an external service. Once the
decision is made by the child decision, the result is processed with the fail-fast
strategy. That is, whenever one child decision is negative (NO), we consider the
whole decision as NO without asking the remaining deciders. It's a short
circuit. We do that unless we are debugging the decision, in which case, we
gather all the decisions, including the NO decisions, before returning the final
result. If my explanation is a bit confusing for you, here is the source code of
`AllocationDeciders`, hopefully it makes things clearer.

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

Now if we go further into the structure of the value class `Decision.Multi`,
you can see that a multi-decision is a decision container, it contains
multiple child decisions inside it:

```java
    public static class Multi extends Decision implements ToXContentFragment {

        private final List<Decision> decisions = new ArrayList<>();
        ...
    }
```

Unlike single decisions, it does not have label and explanation. More precisely,
getting a label from a multi-decision returns `null` and getting an explanation
from a multi-decision throws an unsupported operation exception. In my opinion,
the `Decision` base class is too vague and we shouldn't return null or raising an
exception. But that's a design choice and probably does not matter as this system
is internal to Elasticsearch.

## Lifecycle

_When are deciders created? I.e. where are deciders positioned in the lifecycle of
an Elasticsearch cluster?_

When starting a new node, the class `Bootstrap` is called. Before starting the
node, it creates a new `Node` instance, which creates and addes a list of modules.
One of these modules is
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

Now another question is: _when we update a setting of a cluster, do we need to restart
the node to take the new value into account?_

I don't think so because I handled such
cases many times in production and it never requires a restart. These settings are dynamic.
When looking into Elasticsearch's source code, such as awareness allocation decider and
disk threadhold decider, we can find out that the settings are passed
from the constructor of the deciders.

```java
public AwarenessAllocationDecider(Settings settings, ClusterSettings clusterSettings) { ... }
```

```java
public DiskThresholdDecider(Settings settings, ClusterSettings clusterSettings) {
    this.diskThresholdSettings = new DiskThresholdSettings(settings, clusterSettings);
    assert Version.CURRENT.major < 9 : "remove enable_for_single_data_node in 9";
    this.enableForSingleDataNode = ENABLE_FOR_SINGLE_DATA_NODE.get(settings);
}
```

And the decider is subscribed to the settings update. In other words, when a setting is updated,
the setting used by the decider is updated as well. Here is the code for subscribing the settings
for disk thresholds (low watermark, high watermark, flood-stage) in class `DiskThresholdSettings`:

```java
public DiskThresholdSettings(Settings settings, ClusterSettings clusterSettings) {
    ...
    clusterSettings.addSettingsUpdateConsumer(CLUSTER_ROUTING_ALLOCATION_LOW_DISK_WATERMARK_SETTING, this::setLowWatermark);
    clusterSettings.addSettingsUpdateConsumer(CLUSTER_ROUTING_ALLOCATION_HIGH_DISK_WATERMARK_SETTING, this::setHighWatermark);
    clusterSettings.addSettingsUpdateConsumer(CLUSTER_ROUTING_ALLOCATION_DISK_FLOOD_STAGE_WATERMARK_SETTING, this::setFloodStage);
    ...
}
```

where you can see that when those cluster settings are upated, the class `DiskThresholdSettings`
sets the new value into it's instance.

## Testing

There are three types of testing related to the deciders: 1) the tests for the
single deciders which makes one decision at a time; 2) the tests for the root
decider which gathers information from multiple single decisions and make a
multi-decision; 3) other services that depend on deciders since deciders are
part of the cluster module. And ... we are going to discuss three of them in today's
article :)

**Testing a single-decision decider.** To understand this, let's take the disk threshold
decider as an example (`DiskThresholdDeciderTests`). As we saw in the previous sections "Responsibility" and
"Making Decisions", the allocation is made using the index metadata, shard routing,
routing allocation, etc. All these data are fetched eagerly and passed as input parameters.
Therefore, it makes the test quite simple: we just need to provide these information to
a decider without mocking anything as there are no external calls. More precisely, in the tests, we

* prepare settings that are relevant to this decider
* create a decider instance using these settings
* prepare a cluster state (including index metadata, shard routing, etc) which are
  necessary for making the decision
* create an allocation service which encapsulates the decider under test
* require re-routing the shards by calling the service, which asks the deciders to make
  decision behind the screen
* finally assert the result

... and then iterate through other scenarios.

**Testing a multi-decision decider.** To understand this, let's take
`AllocationDecidersTests` as example. In this test suite, it tests the debug mode and
the early termination. Because these expectations are not tight to any specific child
deciders, the set up simply uses some mocked deciders instantiated as anonymous classes.

```java
final AllocationDeciders allocationDeciders = new AllocationDeciders(Arrays.asList(
    new AllocationDecider() { ... }, new AllocationDecider() { ... }));
```

**Testing a service that depends on the deciders.** In the case, the testing is more complex
to set up because we need to prepare the deciders. Depending on the needs of that service,
it may plug some decider during the setup phase. The easiest setup is a no-operation setup,
which requires an empty all-deciders decider.

```java
public class ClusterAllocationExplainActionTests extends ESTestCase {

    private static final AllocationDeciders NOOP_DECIDERS
        = new AllocationDeciders(Collections.emptyList());
    ...
}
```

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
- Emily Chang, ["How to resolve unassigned shards in
  Elasticsearch"](https://www.datadoghq.com/blog/elasticsearch-unassigned-shards/),
  _Datadog_, 2019.
