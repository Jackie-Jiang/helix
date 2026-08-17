<!---
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

<head>
  <title>Tutorial - Mixed Mode: Pinning Partition Placement in FULL_AUTO</title>
</head>

## [Helix Tutorial](./Tutorial.html): Mixed Mode - Pinning Partition Placement in FULL_AUTO

In FULL_AUTO rebalance mode, Helix controls both the placement and the state of every replica. This is the recommended mode for most applications, but sometimes an operator needs to temporarily take manual control of the placement of a few specific partitions — for example, to isolate a hot partition on dedicated nodes, to work around a hardware issue, or to hold a partition in place during a migration.

Mixed mode makes this possible without changing the rebalance mode of the resource. A resource stays in FULL_AUTO, but selected partitions can be given a *user-defined preference list*. Those partitions behave as if they were SEMI_AUTO: you control the placement, and Helix still controls the state assignment (e.g., which replica is MASTER and which is SLAVE). All other partitions of the resource remain fully managed by Helix.

```
                     | Pinned partitions  | All other partitions |
            ---------------------------------------------------- |
   LOCATION |        APP (you)           |        HELIX          |
            ---------------------------------------------------- |
      STATE |        HELIX               |        HELIX          |
            ------------------------------------------------------
```

### How It Works

The user-defined preference lists are stored in the **ResourceConfig** of the resource (as list fields), not in the IdealState. On every rebalance pipeline run, the rebalancer (both the DelayedAutoRebalancer and the WAGED rebalancer support this) first computes the normal full-auto assignment for all partitions, and then overwrites the preference lists of any partitions that appear in the ResourceConfig with the user-defined lists.

This has a few important properties:

* The IdealState remains FULL_AUTO. You do not need to switch the resource to SEMI_AUTO or convert preference lists for every partition.
* Pinning is per-partition. Only the partitions listed in the ResourceConfig are pinned; the rest continue to be balanced automatically.
* Pinning is fully reversible. Removing a partition's entry from the ResourceConfig returns that partition to full-auto management on the next rebalance, without disturbing the placement of any other partition.
* State assignment is still done by Helix, following the state model and the order of the instances in your preference list (the first instance in the list is preferred for the top state).

### Pinning Partitions

Suppose resource `MyResource` is in FULL_AUTO mode and you want to pin `MyResource_0` to node1 (preferred for the top state) and node2, while leaving all other partitions fully auto.

Using the Java API:

```
ConfigAccessor configAccessor = new ConfigAccessor(zkClient);

Map<String, List<String>> userDefinedPreferenceLists = new HashMap<>();
userDefinedPreferenceLists.put("MyResource_0", Arrays.asList("node1_12918", "node2_12918"));

ResourceConfig resourceConfig =
    new ResourceConfig.Builder("MyResource")
        .setPreferenceLists(userDefinedPreferenceLists)
        .build();
configAccessor.setResourceConfig(clusterName, "MyResource", resourceConfig);
```

If the resource already has a ResourceConfig, update it instead of overwriting it:

```
ResourceConfig resourceConfig = configAccessor.getResourceConfig(clusterName, "MyResource");
Map<String, List<String>> lists = resourceConfig.getPreferenceLists();
lists.put("MyResource_0", Arrays.asList("node1_12918", "node2_12918"));
resourceConfig.setPreferenceLists(lists);
configAccessor.setResourceConfig(clusterName, "MyResource", resourceConfig);
```

Using the Helix REST 2.0 service, POST the preference lists as list fields of the ResourceConfig:

```
$ curl -X POST -H "Content-Type: application/json" \
    http://localhost:8100/admin/v2/clusters/MyCluster/resources/MyResource/configs?command=update \
    -d '
{
  "id" : "MyResource",
  "simpleFields" : {},
  "listFields" : {
    "MyResource_0" : ["node1_12918", "node2_12918"]
  },
  "mapFields" : {}
}'
```

After the next rebalance, the IdealState of `MyResource` will show your preference list for `MyResource_0`, and Helix will move the replicas accordingly and assign states following the list order. All other partitions keep their automatically computed placement.

### Unpinning Partitions

To return a partition to full-auto management, remove its entry from the ResourceConfig preference lists:

```
ResourceConfig resourceConfig = configAccessor.getResourceConfig(clusterName, "MyResource");
Map<String, List<String>> lists = resourceConfig.getPreferenceLists();
lists.remove("MyResource_0");
resourceConfig.setPreferenceLists(lists);
configAccessor.setResourceConfig(clusterName, "MyResource", resourceConfig);
```

Or via REST, delete the list field:

```
$ curl -X POST -H "Content-Type: application/json" \
    http://localhost:8100/admin/v2/clusters/MyCluster/resources/MyResource/configs?command=delete \
    -d '
{
  "id" : "MyResource",
  "simpleFields" : {},
  "listFields" : {
    "MyResource_0" : []
  },
  "mapFields" : {}
}'
```

On the next rebalance, Helix recomputes the placement of the unpinned partition. Partitions that are still pinned keep their user-defined lists, and the placement of partitions that were never pinned is not affected.

### Best Practices and Caveats

* **Validate the instances in the preference list.** Helix applies the user-defined lists as-is. If a listed instance is offline, disabled, or unable to host the replica, the replica may end up in ERROR state or the partition may be under-replicated. Make sure the instances exist, are live, and are healthy before pinning.
* **Match the replica count.** The length of the preference list should equal the resource's replica count. A shorter list results in fewer replicas for that partition.
* **List order matters.** Instances earlier in the list are preferred for higher-priority states (e.g., the first instance is preferred for MASTER/LEADER).
* **Pinned partitions are excluded from rebalancing.** While pinned, a partition will not move even if the cluster topology changes (nodes added/removed) or if its placement becomes unbalanced. Constraints such as delayed rebalancing and min-active-replica maintenance do not apply to pinned partitions — the user-defined list is authoritative.
* **Use it as a temporary, surgical tool.** Mixed mode is intended for operational overrides on a small number of partitions. If you need permanent manual placement of all partitions, use SEMI_AUTO mode instead. Remember to unpin partitions once the operational issue is resolved so Helix can balance them again.
* **Works with both rebalancers.** User-defined preference lists are honored by the DelayedAutoRebalancer (with CRUSH, CrushED, and other strategies) and by the WAGED rebalancer.
