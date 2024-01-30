# Cluster. Zookeeper

Clickhouse uses additional service for cluster nodes synchronization of your choice

* ZooKeeper
* ClickhouseKeeper

This service can be deployed as standalone or in another cluster, like Clickhouse itself. So in the end we will have 2 clusters. One for Clickhouse data and one for Clickhouse cluster metadata.

\*Keeper service\cluster is only needed if you use Replication type of tables in Clickhouse. For pure sharding it is not needed.&#x20;

So. Which one to choose?

According to Clickhouse company, their ClickhouseKeeper is better in any way. Simpler, stabler, faster, etc...but. It has the main drawback. It is new and does not have much documentation, track record and community support yet.

Altinity and other seasoned Clickhouse tamers are not yet advise for ClickhouseKeeper as default. So, you could try it, but i will use Zookeeper here. You can convert it to ClickhouseKeeper later.

**Installation**

It is better to start with single Zookeeper server. Not cluster. It is simple to upscale single node to multiply nodes later if you need fail-over property of a cluster.

```bash
apt update && apt install wget -y

#3.8 is the latest supported version by 23.8 lts CH
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.8.3/apache-zookeeper-3.8.3-bin.tar.gz

#Create directory and extract
mkdir /opt/zookeeper && tar -xvf apache-zookeeper-3.8.3-bin.tar.gz  -C /opt/zookeeper --strip-components=1

#Create data, tlogs dirs
mkdir -p /data/zookeeper/data
mkdir -p /data/zookeeper/datalogs

#Create user
useradd zk -m
usermod --shell /bin/bash zk
usermod -aG sudo zk

#Set rights
chown zk:zk /data/zookeeper/data
chown zk:zk /data/zookeeper/datalogs
chown zk:zk -R /opt/zookeeper

#Utils add to bash
echo 'export PATH="/opt/zookeeper/bin:$PATH"' >> /home/zk/.bashrc
```
