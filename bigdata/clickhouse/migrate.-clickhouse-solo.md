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

```sql
CREATE TABLE opensky
(
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
```

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
