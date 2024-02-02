# Cluster. Clickhouse

Firs of all you should decide what amount of nodes in cluster should be and what types of nodes.

You can have Replicated, Sharded and both nodes.

Replicated nodes copy content written to each of them to another Replicated node in the same shard (and you will always have at least 1 default shard). It is like Raid 1 for tables

Shards are nodes that have half of each sharded table rows. So it is like Raid 0 but for tables.

The base scenario is to have 2 Replicated nodes in 2 shard. So 4 nodes total. That way it will work more like Raid 10. You can lost 1 Replica in each shard without loss of data. So 2 nodes can fail.

Any Replicated table to be writable should have Zookeeper cluster connected to store metadata about tables being synced between Replicas.

You should have Zookeeper cluster installed beforehand or Replicated tables will start as read only.

