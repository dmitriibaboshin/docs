# Settings. Performance

**File Descriptors.**

By default Clickhouse service set high ulimit

```bash
sudo systemctl show clickhouse-server.service |grep NOFILE
```

As you can see

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

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

```
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

You can change it with&#x20;

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



**Tests and compare.**

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
