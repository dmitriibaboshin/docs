# Backup. From Solo

Right now there are several methods to backup clickhouse.

The most tested is Altinity clickhouse-backup. It is not official project, but most tested and with best functionality on the time of writing.

Other options are native Backup (backups only data, not acl and configs) rsync copy with manual ALTER FREEZE.

For easy incremental backups with clickhouse-backup you'll need external S3 storage.&#x20;

If you want to use local storage with incremental backups you'll have to use additional tools like restic, kopia and pair them with clickhouse-backup.&#x20;

Firsts, download and install fresh binary.

```bash
#Download
wget https://github.com/Altinity/clickhouse-backup/releases/download/v2.4.32/clickhouse-backup-linux-amd64.tar.gz

#Install
tar -xf clickhouse-backup-linux-amd64.tar.gz
sudo install -o root -g root -m 0755 build/linux/amd64/clickhouse-backup /usr/local/bin

#Check
/usr/local/bin/clickhouse-backup -v
```

Now generate and edit config

```bash
#Make directory
mkdir /etc/clickhouse-backup
chown root:clickhouse /etc/clickhouse-backup

#Generate Config
clickhouse-backup default-config > /etc/clickhouse-backup/config.yml

#Check
vi /etc/clickhouse-backup/config.yml
```

Now you should change 3 sections in config with your data, assuming you CH is default solo paths and user.&#x20;

```yaml
general:
    remote_storage: s3
    ...
    backups_to_keep_local: 1
    backups_to_keep_remote: 5
    ...
clickhouse:
    username: default
    password: ""
    config_dir: /etc/clickhouse-server/
s3:
    access_key: "your key QWERTY123"
    secret_key: "your key QWERTY123"
    bucket: "ch-backup-bucket-solo"
    endpoint: "https://api.minio.tstlab.xyz"
    ...
    disable_cert_verification: true
    ...
```

Now you can check if backup is working

```bash
#Check tables that clickhouse-backup will consider in CH
clickhouse-backup tables

#Local full backup
clickhouse-backup create local_test_backup --rbac --configs
```

You can see that backup was created. Notice that shadow directory (used as intermediary) is auto cleaned after backup.

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

No let's create remote backup

```bash
#Clean previous remote broken backups
clickhouse-backup clean_remote_broken

#Create full backup
clickhouse-backup create_remote remote_test_backup --rbac --configs
```

Broken backup can occur due link failure, or while backing up default empty RBAC config without policies, etc. It looks like this

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

By default create\_remote will create 2 backup. Local and remote backups

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

You can delete either or set limits in clickhouse config. 0 is unlim

```yaml
    backups_to_keep_local: -1
    backups_to_keep_remote: 18
```

Now let's create cron jobs for full backups and incremental.&#x20;

Full backup script

```bash
#!/bin/bash
BACKUP_NAME=full_week_1_shard_$(clickhouse-client -q "SELECT getMacro('shard')")

clickhouse-backup \
create_remote \
$BACKUP_NAME >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

exit_code=$?

if [[ $exit_code != 0 ]]; then
  echo "clickhouse-backup create-remote $BACKUP_NAME FAILED and return $exit_code exit code"
  exit $exit_code
fi

```

And incremental

```bash
#!/bin/bash
BACKUP_NAME=incremental_week_1_shard_$(clickhouse-client -q "SELECT getMacro('shard')")

clickhouse-backup \
create_remote \
--diff-from-remote=full_week_1_shard_$(clickhouse-client -q "SELECT getMacro('shard')") \
$BACKUP_NAME >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

exit_code=$?

if [[ $exit_code != 0 ]]; then
  echo "clickhouse-backup create-remote $BACKUP_NAME FAILED and return $exit_code exit code"
  exit $exit_code
fi

```

Now create crontab like this&#x20;

```bash
#Full backup
0 0 * 1 * /usr/bin/local/full_week_1.sh
#Incremental backup
0 0 * 2-7 * /usr/bin/local/incremental_week_1.sh

#Full backup
0 0 * 8 * /usr/bin/local/full_week_2.sh
#Incremental backup
0 0 * 9-14 * /usr/bin/local/incremental_week_2.sh

#Full backup
0 0 * 15 * /usr/bin/local/full_week_3.sh
#Incremental backup
0 0 * 16-21 * /usr/bin/local/incremental_week_3.sh

#Full backup last weeks
0 0 * 22 * /usr/bin/local/full_week_4.sh
#Incremental backup
0 0 * 23-31 * /usr/bin/local/incremental_week_4.sh
```

Change week number in script accordingly
