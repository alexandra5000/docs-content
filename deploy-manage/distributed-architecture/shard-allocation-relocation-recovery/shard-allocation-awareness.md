---
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/shard-allocation-awareness.html
applies_to:
  stack:
  deployment:
    self:
products:
  - id: elasticsearch
---

# Shard allocation awareness [shard-allocation-awareness]

You can use custom node attributes as *awareness attributes* to enable {{es}} to take your physical hardware configuration into account when allocating shards. If {{es}} knows which nodes are on the same physical server, in the same rack, or in the same zone, it can distribute the primary shard and its replica shards to minimize the risk of losing all shard copies in the event of a failure.

When shard allocation awareness is enabled with the `cluster.routing.allocation.awareness.attributes` setting, shards are only allocated to nodes that have values set for the specified awareness attributes. If you use multiple awareness attributes, {{es}} considers each attribute separately when allocating shards.

::::{note}
The number of attribute values determines how many shard copies are allocated in each location. If the number of nodes in each location is unbalanced and there are a lot of replicas, replica shards might be left unassigned.
::::


::::{tip}
Learn more about [designing resilient clusters](../../production-guidance/availability-and-resilience/resilience-in-larger-clusters.md).
::::


## Enabling shard allocation awareness [enabling-awareness]

To enable shard allocation awareness:

1. Specify the location of each node with a [custom node attribute](elasticsearch://reference/elasticsearch/configuration-reference/node-settings.md#custom-node-attributes). For example, if you want {{es}} to distribute shards across different racks, you might use an awareness attribute called `rack_id`.

    You can set custom attributes in two ways:

    * By editing the [`elasticsearch.yml`](/deploy-manage/stack-settings.md) config file:

        ```yaml
        node.attr.rack_id: rack_one
        ```

    * Using the `-E` command line argument when you start a node:

        ```sh
        ./bin/elasticsearch -Enode.attr.rack_id=rack_one
        ```

2. Tell {{es}} to take one or more awareness attributes into account when allocating shards by setting `cluster.routing.allocation.awareness.attributes` in **every** master-eligible node’s [`elasticsearch.yml`](/deploy-manage/stack-settings.md) config file.

    ```yaml
    cluster.routing.allocation.awareness.attributes: rack_id <1>
    ```

    1. Specify multiple attributes as a comma-separated list.


    You can also use the [cluster-update-settings](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-put-settings) API to set or update a cluster’s awareness attributes:

    ```console
    PUT /_cluster/settings
    {
      "persistent" : {
        "cluster.routing.allocation.awareness.attributes" : "rack_id"
      }
    }
    ```


With this example configuration, if you start two nodes with `node.attr.rack_id` set to `rack_one` and create an index with 5 primary shards and 1 replica of each primary, all primaries and replicas are allocated across the two node.

:::{image} /deploy-manage/images/elasticsearch-reference-shard-allocation-awareness-one-rack.png
:alt: All primaries and replicas are allocated across two nodes in the same rack
:title: All primaries and replicas allocated across two nodes in the same rack
:::

If you add two nodes with `node.attr.rack_id` set to `rack_two`, {{es}} moves shards to the new nodes, ensuring (if possible) that no two copies of the same shard are in the same rack.

:::{image} /deploy-manage/images/elasticsearch-reference-shard-allocation-awareness-two-racks.png
:alt: Primaries and replicas are allocated across four nodes in two racks with no two copies of the same shard in the same rack
:title: Primaries and replicas allocated across four nodes in two racks, with no two copies of the same shard in the same rack
:::

If `rack_two` fails and takes down both its nodes, by default {{es}} allocates the lost shard copies to nodes in `rack_one`. To prevent multiple copies of a particular shard from being allocated in the same location, you can enable forced awareness.


## Forced awareness [forced-awareness]

By default, if one location fails, {{es}} spreads its shards across the remaining locations. This might be undesirable if the cluster does not have sufficient resources to host all its shards when one location is missing.

To prevent the remaining locations from being overloaded in the event of a whole-location failure, specify the attribute values that should exist with the `cluster.routing.allocation.awareness.force.*` settings. This will mean that {{es}} will prefer to leave some replicas unassigned in the event of a whole-location failure instead of overloading the nodes in the remaining locations.

For example, if you have an awareness attribute called `zone` and configure nodes in `zone1` and `zone2`, you can use forced awareness to make {{es}} leave half of your shard copies unassigned if only one zone is available:

```yaml
cluster.routing.allocation.awareness.attributes: zone
cluster.routing.allocation.awareness.force.zone.values: zone1,zone2 <1>
```

1. Specify all possible `zone` attribute values.


With this example configuration, if you have two nodes with `node.attr.zone` set to `zone1` and an index with `number_of_replicas` set to `1`, {{es}} allocates all the primary shards but none of the replicas. It will assign the replica shards once nodes with a different value for `node.attr.zone` join the cluster. In contrast, if you do not configure forced awareness, {{es}} will allocate all primaries and replicas to the two nodes even though they are in the same zone.


