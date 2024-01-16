# Settings. Initial

**File Descriptors.**

By default Clickhouse service set high ulimit

```bash
sudo systemctl show clickhouse-server.service |grep NOFILE
```

As you can see

<figure><img src="../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

It can be increased by service edit if it is less than 100000

```bash
sudo systemctl edit clickhouse-server.service
```

Then pass

```
[Service]
LimitNOFILE=500001
LimitNOFILESoft=500001
```

And check

```bash
sudo systemctl show clickhouse-server.service |grep NOFILE
```

**Max Memory Usage**

By default there is no need to setup **max\_server\_memory\_usage.** It is calculated as 90% of total memory. Giving 10% for system needs.

But if you have less than 40GB of RAM it is good thing to give system fixed 4GB of RAM.

Create settings file and restart service.

<pre class="language-bash"><code class="lang-bash"><strong>sudo nano /etc/clickhouse-server/config.d/memory_settings.xml
</strong></code></pre>

And pass RAM-4GB (in my case it is 32GB RAM on server)

```xml
<clickhouse>
<!-- Change default 90% memory allocation -->
    <max_server_memory_usage>28G</max_server_memory_usage>
</clickhouse>
```

Restart service

```bash
sudo systemctl restart clickhouse-server.service
```

And check setting in ch client

```sql
SELECT * FROM system.server_settings WHERE name='max_server_memory_usage';
```

**Shared Memory.**

Check if kernel has big kernel.shmmax. It is in bytes and must be greater than 1GB.

```bash
sysctl kernel.shmmax
```

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

You can change it with

```bash
sudo nano /etc/sysctl.conf
```

Than add

```
kernel.shmmax = VALUE
```

Than apply

```bash
sudo sysctl -p
```

**Limit Disk Space.**

You should limit disk space used if disk is used by another processes too.

For that we should create disk policy.

Create additional config

```bash
sudo nano /etc/clickhouse-server/config.d/disks_settings.xml
```

Add this content

```xml
<clickhouse>
<storage_configuration>
<!-- Define disk to be used -->
    <disks>
        <default> <!-- Main Disk -->
            <keep_free_space_bytes>2147483648</keep_free_space_bytes>
        </default>
        <sdb> <!-- Second Disk -->
            <path>/mnt/disks/sdb/clickhouse/</path>
            <keep_free_space_bytes>2147483648</keep_free_space_bytes>
        </sdb>
    </disks>
    
</storage_configuration>
</clickhouse>
```

Restart service and check disks in CH

```sql
SELECT
    name,
    path,
    formatReadableSize(free_space) AS free,
    formatReadableSize(total_space) AS total,
    formatReadableSize(keep_free_space) AS reserved
FROM system.disks
```

Now CH now about disks

<figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

Create Policy

```bash
sudo nano /etc/clickhouse-server/config.d/disks_policy.xml
```

Paste policy for single disk

```xml
<clickhouse>
<storage_configuration>

    <policies>
        <single_disk_policy> <!-- policy name -->
            <volumes>
                <sdb_volume> <!-- volume name -->
                    <disk>sdb</disk>
                </sdb_volume>
            </volumes>
        </single_disk_policy>
    </policies>
    
</storage_configuration>
</clickhouse>
```

Clickhouse will not migrate dbs\tables to another disk! You should create new tables with new settings or manually copy and modify existent (not in this manual) tables.

```sql
CREATE TABLE trips_sdb (
    trip_id             UInt32,
    pickup_datetime     DateTime,
    dropoff_datetime    DateTime,
    pickup_longitude    Nullable(Float64),
    pickup_latitude     Nullable(Float64),
    dropoff_longitude   Nullable(Float64),
    dropoff_latitude    Nullable(Float64),
    passenger_count     UInt8,
    trip_distance       Float32,
    fare_amount         Float32,
    extra               Float32,
    tip_amount          Float32,
    tolls_amount        Float32,
    total_amount        Float32,
    payment_type        Enum('CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4, 'UNK' = 5),
    pickup_ntaname      LowCardinality(String),
    dropoff_ntaname     LowCardinality(String)
)
ENGINE = MergeTree
PRIMARY KEY (pickup_datetime, dropoff_datetime)
SETTINGS storage_policy = 'single_disk_policy';
```

Insert data in new table

<pre class="language-sql"><code class="lang-sql"><strong>INSERT INTO trips_sdb
</strong>SELECT
    trip_id,
    pickup_datetime,
    dropoff_datetime,
    pickup_longitude,
    pickup_latitude,
    dropoff_longitude,
    dropoff_latitude,
    passenger_count,
    trip_distance,
    fare_amount,
    extra,
    tip_amount,
    tolls_amount,
    total_amount,
    payment_type,
    pickup_ntaname,
    dropoff_ntaname
FROM s3(
    'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_{0..2}.gz',
    'TabSeparatedWithNames'
);
</code></pre>

Check parts location

```sql
SELECT table, disk_name, path
FROM system.parts
WHERE table LIKE 'trips%'
```

New table is on another disk

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

**Logging Setup**

Create logging config and dir

```bash
sudo nano /etc/clickhouse-server/config.d/logging_settings.xml;
mkdir /mnt/disks/sdb/clickhouse_logs;
chmod 755 /mnt/disks/sdb/clickhouse_logs;
```

Paste the following content

```xml
<clickhouse>
    <logger>
        <!-- Trace is maximum prod level supported -->
        <level>warning</level>
        <log>/mnt/disks/sdb/clickhouse_logs/clickhouse-server.log</log>
        <errorlog>/mnt/disks/sdb/clickhouse_logs/clickhouse-server.err.log</errorlog>
        <!-- Consider limits for DBs on disk to have this logs space available -->
        <!-- Size is for each log type-->
        <size>500M</size>
        <count>3</count>
        <!-- Comment this if you want automatic console output (when run as not daemon) -->
        <console>0</console>
        <!-- Enable Json log format. Comment <formatting> to disable -->
        <formatting>
            <type>json</type>
            <names>
            <!-- Comment not needed sections -->
                <date_time>date_time</date_time>
                <thread_name>thread_name</thread_name>
                <thread_id>thread_id</thread_id>
                <level>level</level>
                <query_id>query_id</query_id>
                <logger_name>logger_name</logger_name>
                <message>message</message>
                <source_file>source_file</source_file>
                <source_line>source_line</source_line>
            </names>
        </formatting>
    </logger>
</clickhouse>
```

**Misc.**

Create other settings.

```bash
sudo nano /etc/clickhouse-server/config.d/misc_settings.xml
```

Put this in

```xml
<clickhouse>
    <!-- Change time zone for time stamps-->
    <timezone>Europe/Moscow</timezone>
    <!-- Enable listening external connections-->
    <listen_host>0.0.0.0</listen_host>
</clickhouse>
```

**Additional Users.**

By default there is superuser default. But. You can create roles and new users. The old way is to do it with users.d/file method. This files are constantly monitored by clickhouse and applied. You can do the same by SQL commands inside clickhouse client. But files have priority.

Create file

```bash
sudo nano /etc/clickhouse-server/users.d/robocop.xml;
```

Generate password

```bash
PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
```

You will get random password and its hash

<figure><img src="../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

And pass this content and use hash to hide password in config.

```xml
<clickhouse>
<users>
    <!-- If user name was not specified, 'default' user is used. -->
    <robocop>
        <password_sha256_hex>09c3ce5c8cac4081addadf257b27bc201f18cb519a12a6ffbea9911b3e85acb1</password_sha256_hex>
        <!-- Enable this user management with SQL commands -->
        <access_management>1</access_management>
        <!-- Allow user to connect from any network -->
        <networks incl="networks" replace="replace">
        <ip>::/0</ip>
        </networks>

        <!-- Create profile with non default clickhouse settings -->
        <profile>robots_profile</profile>

        <!-- Limit query amounts etc. -->
        <quota>robots_quota</quota>

        <default_database>robots_db</default_database>

        <!-- Allow only rows with id=1000 for database and table -->
        <databases>
            <robots_db>
                <test_table>
                    <filter>id = 1000</filter>
                </test_table>
            </robots_db>
        </databases>
        
        <!-- Rights settings. This one is admin -->
        <grants>
            <query>GRANT SELECT ON system.*</query>
        </grants>
        
    </robocop>
    <!-- Other users settings, same as block above -->
</users>
</clickhouse>
```

Now create profiles and quota files

```bash
sudo nano /etc/clickhouse-server/users.d/robots_profile.xml;
```

Content

```xml
<clickhouse>
<profiles>
    <!-- Default settings. -->
    <default>
    </default>

    <!-- Profile that allows only read queries. -->
    <readonly>
        <readonly>1</readonly>
    </readonly>

    <robots_profile>
        <max_rows_to_read>1000000000</max_rows_to_read>
        <max_bytes_to_read>100000000000</max_bytes_to_read>

        <max_rows_to_group_by>1000000</max_rows_to_group_by>
        <group_by_overflow_mode>any</group_by_overflow_mode>

        <max_rows_to_sort>1000000</max_rows_to_sort>
        <max_bytes_to_sort>1000000000</max_bytes_to_sort>

        <max_result_rows>100000</max_result_rows>
        <max_result_bytes>100000000</max_result_bytes>
        <result_overflow_mode>break</result_overflow_mode>

        <max_execution_time>600</max_execution_time>
        <min_execution_speed>1000000</min_execution_speed>
        <timeout_before_checking_execution_speed>15</timeout_before_checking_execution_speed>

        <max_columns_to_read>25</max_columns_to_read>
        <max_temporary_columns>100</max_temporary_columns>
        <max_temporary_non_const_columns>50</max_temporary_non_const_columns>

        <max_subquery_depth>2</max_subquery_depth>
        <max_pipeline_depth>25</max_pipeline_depth>
        <max_ast_depth>50</max_ast_depth>
        <max_ast_elements>100</max_ast_elements>

        <max_sessions_for_user>4</max_sessions_for_user>
        <readonly>1</readonly>
    </robots_profile>
</profiles>
</clickhouse>
```

Quotas file

```bash
sudo nano /etc/clickhouse-server/users.d/robots_quota.xml;
```

Content

```xml
<clickhouse>
<quotas>
    <!-- Name of quota. -->
    <default>
        <!-- Limits for time interval. You could specify many intervals with different limits. -->
        <interval>
            <!-- Length of interval. -->
            <duration>3600</duration>

            <!-- No limits. Just calculate resource usage for time interval. -->
            <queries>0</queries>
            <errors>0</errors>
            <result_rows>0</result_rows>
            <read_rows>0</read_rows>
            <execution_time>0</execution_time>
        </interval>
    </default>

    <robots_quota>
    <!-- Limit load from user for 1h and 24h -->
        <interval>
            <duration>3600</duration>

            <queries>1000</queries>
            <query_selects>100</query_selects>
            <query_inserts>100</query_inserts>
            <errors>100</errors>
            <result_rows>1000000000</result_rows>
            <read_rows>100000000000</read_rows>
            <execution_time>900</execution_time>
        </interval>
        <interval>
            <duration>86400</duration>

            <queries>10000</queries>
            <query_selects>10000</query_selects>
            <query_inserts>10000</query_inserts>
            <errors>1000</errors>
            <result_rows>5000000000</result_rows>
            <read_rows>500000000000</read_rows>
            <execution_time>7200</execution_time>
        </interval>
    </robots_quota>
</quotas>
</clickhouse>
```

**Performance Tests.**

Let's run some benchmarks on clickhouse.

This first test is for default ch binaries on empty machine without ch installed.

Download test and clickhouse binary with a script. Set the db downloaded size from 255 sets to 50

```bash
#Right now script is written for root and root dir
sudo su - 
cd /

#Download and chmod
wget https://raw.githubusercontent.com/ClickHouse/ClickBench/main/hardware/hardware.sh

#Change size
sed -i 's/0..255/0..50/g' hardware.sh

#Run
chmod +x hardware.sh
./hardware.sh
```

The output should be like this. If no, check paths.

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Now we will modify test to use installed and tuned ch (therefore you should install it now and use tweaks above).

Comment this sections in hardware.sh and change paths to use service and not local binary.

```bash
#clickhouse server >server.log 2>&1 &
#PID=$!
#
#function finish {
#    kill $PID
#    wait
#}
#trap finish EXIT
```

Paths with sed

```bash
sed -i 's|./clickhouse |clickhouse |g' hardware.sh;

sed -i 's|/clickhouse-benchmark/server\.log|/var/log/clickhouse-server/clickhouse-server.log|g' hardware.sh;

#Again change sets size if you downloaded fresh script
sed -i 's/0..255/0..50/g' hardware.sh
```

And run again

```bash
chmod +x hardware.sh
./hardware.sh
```

Compare with new results
