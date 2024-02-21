---
description: >-
  On this page we will try several methods of clickhouse migrations from single
  host to running cluster.
---

# Migrate. Clickhouse Solo

First, let us upload big data set onto our test solo server.

```bash
#Create folder for test data
mkdir clickhouse_test_data
#Download 8GB archives
wget -O- https://zenodo.org/records/7923702 | grep -oP 'https://zenodo.org/records/7923702/files/flightlist_\d+_\d+\.csv\.gz' | xargs wget

```

Now create table

<pre class="language-sql"><code class="lang-sql"><strong>CREATE TABLE opensky
</strong>(
    callsign String,
    number String,
    icao24 String,
    registration String,
    typecode String,
    origin String,
    destination String,
    firstseen DateTime,
    lastseen DateTime,
    day DateTime,
    latitude_1 Float64,
    longitude_1 Float64,
    altitude_1 Float64,
    latitude_2 Float64,
    longitude_2 Float64,
    altitude_2 Float64
) ENGINE = MergeTree ORDER BY (origin, destination, callsign);
</code></pre>

Import data (you'll need 32GB RAM on clickhouse server).

```bash
ls -1 flightlist_*.csv.gz | \
xargs -P100 -I{} bash -c 'gzip -c -d "{}" | \
clickhouse-client --date_time_input_format best_effort --query "INSERT INTO opensky FORMAT CSVWithNames"'
```

Now checks in client

```sql
SELECT count() FROM opensky;

SELECT formatReadableSize(total_bytes) FROM system.tables WHERE name = 'opensky';
```

Results

<figure><img src="../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

If you want to make migration consistent, you should block client SQL writes onto solo server. For that you should set "readonly" flag for corresponding SQL users. (be careful with default, it also used in dist tables and replicas). Therefore

```sql
#Make user readonly for new sessions
ALTER USER ch_user ON CLUSTER ch_cluster SETTINGS readonly=1;

#Optional. Set Idle connection timeout 1 sec. For new sessions. Rollback after
ALTER USER ch_user ON CLUSTER ch_cluster SETTINGS idle_connection_timeout=1;

```

Now you should wait 1h idle while user sessions are closed and proceed.&#x20;

#### Method 1. Select from Remote

This method is good when you want compatibility between different clickhouse version, have small table sizes and reliable connection. Remember that clickhouse does not have atomic transactions.

First you should create tables on your cluster. Replicated and Distributed.

```sql
#Check that remote solo click node is available
SELECT count() FROM remote('srv-pve-p-u-big3clicksolo-0.tstlab.xyz', default, opensky, 'default', '');

#Create tables
CREATE TABLE IF NOT EXISTS default.opensky_local ON CLUSTER sharded_replicas (
    callsign String,
    number String,
    icao24 String,
    registration String,
    typecode String,
    origin String,
    destination String,
    firstseen DateTime,
    lastseen DateTime,
    day DateTime,
    latitude_1 Float64,
    longitude_1 Float64,
    altitude_1 Float64,
    latitude_2 Float64,
    longitude_2 Float64,
    altitude_2 Float64
    ) ENGINE = ReplicatedMergeTree ('/clickhouse/tables/{cluster}/{shard}/{database}/{table}', '{replica}')
ORDER BY (origin, destination, callsign);

CREATE TABLE IF NOT EXISTS default.opensky ON CLUSTER sharded_replicas
AS default.opensky_local
ENGINE = Distributed(sharded_replicas, default, opensky_local, rand());
```

Next. From one node of a clickhouse cluster run

```sql
INSERT INTO default.opensky SELECT * FROM remote('srv-pve-p-u-clicksolo-0.tstlab.xyz', default, opensky, 'default', '');
```

Now check that you data is loaded and properly sharded. Run on cluster node

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

On my 10GBit channel migration took about 80 seconds and 18GB traffic.&#x20;

You can see above that in distributed table has the same records number as in solo server. But in replicated table number is about a half of that. As should it be when data is sharded onto 2 replica sets.

BTW, if your connection will drop permanently or you will press CTRL C, you will have some rows copied anyway.&#x20;

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

If you have unstable connection you can change default timeout from 300 sec to 1h in query like this

```sql
INSERT INTO default.opensky SETTINGS receive_timeout=3600 SELECT * FROM remote('srv-pve-p-u-big3clicksolo-0.tstlab.xyz', default, opensky, 'default', '');
```

Clickhouse will continue to download data if connection is restored in this time period.
