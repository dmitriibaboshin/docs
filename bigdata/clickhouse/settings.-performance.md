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

