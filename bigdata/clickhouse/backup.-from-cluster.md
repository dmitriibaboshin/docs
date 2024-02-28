# Backup. From Cluster

When you have and CH cluster, you almost certainly will have Replicated tables and Shards.

Shards are like Raid 0 in DBMS when data is split across multiply servers. Therefore we have to backup each peace of data to a separate backup.

Let's assume we have 2 shards and 2 replicas in each shard, so total 4 nodes.

It is best to backup 1st replica in each shard. And if you worry that backup data won't be consistent, you are right. But it is by design in default asynchronous CH nature. But, it is better than nothing.

So we have this config
