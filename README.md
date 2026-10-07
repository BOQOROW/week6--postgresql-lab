# Week 6: Backups, Point-in-Time Recovery, and Replication (PostgreSQL)

Environment: Ubuntu (WSL) on Windows, PostgreSQL 18.6, database `bootcamp`.

## Setup

Created the database and a `students` table with 3 rows:

```
CREATE DATABASE bootcamp;
CREATE TABLE students (id SERIAL PRIMARY KEY, name VARCHAR(50));
INSERT INTO students (name) VALUES ('Boqorow'), ('Amina'), ('Omar');
```

## Step 1: Logical backup and verification

```
mkdir -p ~/backups
pg_dump -Fc -f ~/backups/bootcamp.dump bootcamp
pg_restore --list ~/backups/bootcamp.dump | head
createdb bootcamp_check && pg_restore -d bootcamp_check ~/backups/bootcamp.dump
psql bootcamp_check -c "SELECT * FROM students;"
```

Output:

```
;     dbname: bootcamp
;     TOC Entries: 11
;     Format: CUSTOM
;     Dumped from database version: 18.6 (Ubuntu 18.6-0ubuntu0.26.04.1)

 id |  name
----+---------
  1 | Boqorow
  2 | Amina
  3 | Omar
(3 rows)
```

The restored copy `bootcamp_check` matches the original, so the backup works.

## Step 2: WAL archiving and base backup

```
mkdir -p ~/backups/wal
chmod o+x ~
chmod 777 ~/backups/wal
sudo -u postgres psql -c "ALTER SYSTEM SET wal_level = 'replica';"
sudo -u postgres psql -c "ALTER SYSTEM SET archive_mode = 'on';"
sudo -u postgres psql -c "ALTER SYSTEM SET archive_command = 'cp %p /home/boqorow/backups/wal/%f';"
sudo service postgresql restart
pg_basebackup -D ~/backups/base -Ft -z -Xs -P
```

Output:

```
 archive_mode
--------------
 on

          archive_command
------------------------------------
 cp %p /home/boqorow/backups/wal/%f

39339/39339 kB (100%), 1/1 tablespace

-rw------- 1 boqorow boqorow 221K Oct  7 12:13 backup_manifest
-rw------- 1 boqorow boqorow 5.1M Oct  7 12:13 base.tar.gz
-rw------- 1 boqorow boqorow  17K Oct  7 12:13 pg_wal.tar.gz
```

## Step 3: Simulated disaster and point-in-time recovery

Recorded the time, then deleted the data:

```
psql bootcamp -c "SELECT now();"
```
```
 2026-10-07 12:15:39.019248+03
```

```
psql bootcamp -c "DELETE FROM students;"
psql bootcamp -c "SELECT pg_switch_wal();"
psql bootcamp -c "SELECT count(*) FROM students;"
```
```
 count
-------
     0
```

Recovery:

```
sudo service postgresql stop
sudo mv /var/lib/postgresql/18/main /var/lib/postgresql/18/main_old
sudo mkdir /var/lib/postgresql/18/main
sudo tar -xzf ~/backups/base/base.tar.gz -C /var/lib/postgresql/18/main
sudo tar -xzf ~/backups/base/pg_wal.tar.gz -C /var/lib/postgresql/18/main/pg_wal
sudo chown -R postgres:postgres /var/lib/postgresql/18/main
sudo chmod 700 /var/lib/postgresql/18/main
echo "restore_command = 'cp /home/boqorow/backups/wal/%f %p'" | sudo tee -a /var/lib/postgresql/18/main/postgresql.auto.conf
echo "recovery_target_time = '2026-10-07 12:15:39+03'" | sudo tee -a /var/lib/postgresql/18/main/postgresql.auto.conf
echo "recovery_target_action = 'promote'" | sudo tee -a /var/lib/postgresql/18/main/postgresql.auto.conf
sudo -u postgres touch /var/lib/postgresql/18/main/recovery.signal
sudo service postgresql start
```

Key lines from the PostgreSQL log:

```
LOG:  starting point-in-time recovery to 2026-10-07 12:15:39+03
LOG:  recovery stopping before commit of transaction 776, time 2026-10-07 12:16:29.393387+03
LOG:  archive recovery complete
LOG:  database system is ready to accept connections
```

Verification:

```
psql bootcamp -c "SELECT * FROM students;"
```
```
 id |  name
----+---------
  1 | Boqorow
  2 | Amina
  3 | Omar
(3 rows)
```

Recovery stopped just before the `DELETE`, and all 3 rows came back.

## Step 4: Streaming standby

Removed the recovery settings from the primary so the standby would not inherit them, then created the replication role:

```
sudo -u postgres psql -c "ALTER SYSTEM RESET restore_command;"
sudo -u postgres psql -c "ALTER SYSTEM RESET recovery_target_time;"
sudo -u postgres psql -c "ALTER SYSTEM RESET recovery_target_action;"
sudo -u postgres psql -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'reppass';"
echo "host replication replicator 127.0.0.1/32 md5" | sudo tee -a /etc/postgresql/18/main/pg_hba.conf
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

Built and started the standby on port 5433:

```
pg_basebackup -h 127.0.0.1 -U replicator -D ~/standby -R -P
echo "port = 5433" > ~/standby/postgresql.conf
echo "unix_socket_directories = '/tmp'" >> ~/standby/postgresql.conf
printf "local all all trust\nhost all all 127.0.0.1/32 trust\n" > ~/standby/pg_hba.conf
touch ~/standby/pg_ident.conf
/usr/lib/postgresql/18/bin/pg_ctl -D ~/standby -l ~/standby/log.txt start
psql -h 127.0.0.1 -p 5433 bootcamp -c "SELECT pg_is_in_recovery();"
```
```
 pg_is_in_recovery
-------------------
 t
```

`t` means the server is running as a standby.

## Step 5: Replication health

```
psql bootcamp -c "SELECT application_name, state, pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes FROM pg_stat_replication;"
```
```
 application_name |   state   | lag_bytes
------------------+-----------+-----------
 walreceiver      | streaming |         0
```

Inserted a new row on the primary and checked the standby:

```
psql bootcamp -c "INSERT INTO students (name) VALUES ('Test');"
psql -h 127.0.0.1 -p 5433 bootcamp -c "SELECT * FROM students;"
```
```
 id |  name
----+---------
  1 | Boqorow
  2 | Amina
  3 | Omar
  4 | Test
(4 rows)
```

The new row appeared on the standby, so replication is working with zero lag.

## Notes on differences from the lab text

Ubuntu keeps its configuration in `/etc/postgresql/18/main/`, and WSL does not use `systemctl`, so I used these instead of the lab's version:

- `ALTER SYSTEM` and `postgresql.auto.conf` instead of editing `postgresql.conf` by hand
- `sudo service postgresql ...` instead of `sudo systemctl ...`
- Full paths such as `/home/boqorow/...` instead of `$USER` and `~` inside PostgreSQL settings
- `SELECT pg_switch_wal();` after the delete so the last WAL file was archived
