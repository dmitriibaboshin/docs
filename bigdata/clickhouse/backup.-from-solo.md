# Backup. From Solo

Right now there are several methods to backup clickhouse.

The most tested is Altinity **clickhouse-backup**. It is not official project, but most tested and has most functionality on the time of writing.

Other runtime options are native CH Backup command (backups only data, no RBAC and configs) and rsync copy with manual ALTER FREEZE.

For easy incremental backups with clickhouse-backup you'll need external S3 storage.  Incremental backups created using name comparison of parts folders.

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
    backups_to_keep_local: 3
    backups_to_keep_remote: 32
```

Now let's create cron jobs for full backups and incremental. One set each week of the month.

Full backup script for week 1. Make same for 2-4.

```bash
#!/bin/bash
BACKUP_NAME=week_1_shard_$(clickhouse-client -q "SELECT getMacro('shard')")

#Clean old shadow folder hard links
echo "Clean shadow directory"
clickhouse-backup clean >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

#Clean stuck and broken remote backups
echo "Clean broken remote"
clickhouse-backup clean_remote_broken >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

#Remove old local backup
echo "Delete LOCAL Full backup done last time"
clickhouse-backup \
delete local \
full_$BACKUP_NAME >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

#Remove old remote backup
echo "Delete REMOTE Full backup done last time"
clickhouse-backup \
delete remote \
full_$BACKUP_NAME >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

#Full default backup with schema and server configs
echo "Create LOCAL AND REMOTE Full backup"
clickhouse-backup \
create_remote \
full_$BACKUP_NAME --configs >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

exit_code=$?

if [[ $exit_code != 0 ]]; then
  echo "clickhouse-backup create_remote $BACKUP_NAME FAILED and return $exit_code exit code"
  exit $exit_code
fi
```

And incremental

```bash
#!/bin/bash
BACKUP_NAME_DIFF=incremental_week_1_shard_$(clickhouse-client -q "SELECT getMacro('shard')")_$(date +%Y_%m_%d)

#Clean old shadow folder hard links
echo "Clean shadow directory"
clickhouse-backup clean >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

#Clean stuck and broken remote backups
echo "Clean broken remote"
clickhouse-backup clean_remote_broken >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

#Delete local previous Incremental backup
echo "Delete LOCAL Incremental OLD backup done that day last time"
clickhouse-backup \
delete local \
$BACKUP_NAME_DIFF >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

#Create Incremental backup
echo "Delete REMOTE Incremental OLD backup done that day last time"
clickhouse-backup \
delete remote \
$BACKUP_NAME_DIFF >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

#Create Incremental backup
echo "Create LOCAL AND REMOTE Incremental NEW backup for current day"
clickhouse-backup \
create_remote \
--diff-from-remote=full_week_week_1_shard_$(clickhouse-client -q "SELECT getMacro('shard')") \
$BACKUP_NAME_DIFF >> /data/clickhouse/logs/clickhouse-backup.log 2>&1

exit_code=$?

if [[ $exit_code != 0 ]]; then
  echo "clickhouse-backup create_remote $BACKUP_NAME_DIFF FAILED and return $exit_code exit code"
  exit $exit_code
fi

```

Now create crontab like this&#x20;

```bash
#Full backup
0 0 * 1 * /usr/local/bin/full_week_1.sh
#Incremental backup
0 0 * 2-7 * /usr/local/bin/incremental_week_1.sh

#Full backup
0 0 * 8 * /usr/local/bin/full_week_2.sh
#Incremental backup
0 0 * 9-14 * /usr/local/bin/incremental_week_2.sh

#Full backup
0 0 * 15 * /usr/local/bin/full_week_3.sh
#Incremental backup
0 0 * 16-21 * /usr/local/bin/incremental_week_3.sh

#Full backup last weeks
0 0 * 22 * /usr/local/bin/full_week_4.sh
#Incremental backup
0 0 * 23-31 * /usr/local/bin/incremental_week_4.sh
```

Change week number in script accordingly
