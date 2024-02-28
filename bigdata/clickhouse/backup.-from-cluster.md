# Backup. From Cluster

When you have and CH cluster, you almost certainly will have Replicated tables and Shards.

Shards are like Raid 0 in DBMS when data is split across multiply servers. Therefore we have to backup each peace of data to a separate backup.

Let's assume we have 2 shards and 2 replicas in each shard, so total 4 nodes.

It is best to backup 1st replica in each shard. And if you worry that backup data won't be consistent, you are right. But it is by design in default asynchronous CH nature. But, it is better than nothing.

So we have this config

```xml
<clickhouse>
    <remote_servers replace="true">
        <ch_cluster>
    <secret>cluster-pass123</secret>
                <shard>
            <internal_replication>true</internal_replication>
                        <replica>
                <host>10.10.99.120</host>
                <port>9000</port>
	        </replica>
                        <replica>
                <host>10.10.99.121</host>
                <port>9000</port>
	        </replica>
                    </shard>
                <shard>
            <internal_replication>true</internal_replication>
                        <replica>
                <host>10.10.99.122</host>
                <port>9000</port>
	        </replica>
                        <replica>
                <host>10.10.99.123</host>
                <port>9000</port>
	        </replica>
                    </shard>
            </ch_cluster>
        </remote_servers>
</clickhouse>

```

So, we should setup **clickhouse-backup** on 10.10.99.120 and 10.10.99.122 in this example.

First, install on both.

```bash
#Download
wget https://github.com/Altinity/clickhouse-backup/releases/download/v2.4.32/clickhouse-backup-linux-amd64.tar.gz

#Install
tar -xf clickhouse-backup-linux-amd64.tar.gz
sudo install -o root -g root -m 0755 build/linux/amd64/clickhouse-backup /usr/local/bin

#Check
/usr/local/bin/clickhouse-backup -v
```
