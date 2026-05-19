````markdown
# Postgresql 18 Manual Failover PoC using Docker inside WSL2

Simple PostgreSQL 18 High Availability (HA) Proof of Concept using:

- PostgreSQL 18
- Docker
- WSL2 Ubuntu
- Streaming Replication
- Manual Failover

This PoC is designed for learning PostgreSQL replication concepts without:
- Patroni
- Kubernetes
- Docker Compose

---

# Architecture

```text
                Docker Network : pgnet

        +-----------------------------+
        |         pg-primary          |
        | PostgreSQL 18 PRIMARY       |
        | Port 5434                   |
        +-------------+---------------+
                      |
              Streaming WAL
                      |
        -----------------------------------------
        |                   |                   |
        v                   v                   v

+----------------+  +----------------+  +----------------+
|  pg-standby1   |  |  pg-standby2   |  |  pg-standby3   |
| PostgreSQL 18  |  | PostgreSQL 18  |  | PostgreSQL 18  |
| Port 6001      |  | Port 6002      |  | Port 6003      |
+----------------+  +----------------+  +----------------+
````

---

# Environment

* Ubuntu WSL2
* Docker Engine
* PostgreSQL 18 Docker Image

---

# STEP 1 — Cleanup Old Setup

```bash
docker stop pg-primary pg-standby1 pg-standby2 pg-standby3 2>/dev/null

docker rm pg-primary pg-standby1 pg-standby2 pg-standby3 2>/dev/null

docker volume rm pg_primary_data 2>/dev/null
```

---

# STEP 2 — Create Docker Network

```bash
docker network create pgnet
```

Verify:

```bash
docker network ls
```

You should see:

```text
pgnet
```

---

# STEP 3 — Pull PostgreSQL 18 Image

```bash
docker pull postgres:18
```

---

# STEP 4 — Start PostgreSQL PRIMARY Container

```bash
docker run -d \
--name pg-primary \
--network pgnet \
-p 5434:5432 \
-e POSTGRES_PASSWORD=postgres \
-v pg_primary_data:/var/lib/postgresql \
postgres:18
```

---

# STEP 5 — Verify PRIMARY Running

```bash
docker ps
```

Expected:

```text
pg-primary   Up
```

---

# STEP 6 — Connect PRIMARY Database

```bash
docker exec -it pg-primary psql -U postgres
```

Check version:

```sql
SELECT version();
```

Exit:

```sql
\q
```

---

# STEP 7 — Install vim Editor Inside Container

Enter container:

```bash
docker exec -it pg-primary bash
```

Install vim:

```bash
apt update

apt install -y vim
```

Exit:

```bash
exit
```

---

# STEP 8 — Create Replication User

Connect PostgreSQL:

```bash
docker exec -it pg-primary psql -U postgres
```

Run:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replpass';

SHOW wal_level;

SHOW max_wal_senders;
```

Expected:

```text
replica
```

Exit:

```sql
\q
```

---

# STEP 9 — Configure pg_hba.conf

Enter PRIMARY container:

```bash
docker exec -it pg-primary bash
```

Find data directory:

```bash
psql -U postgres -c "SHOW data_directory;"
```

Expected:

```text
/var/lib/postgresql/18/docker
```

Edit pg_hba.conf:

```bash
vi /var/lib/postgresql/18/docker/pg_hba.conf
```

Add this line at the END:

```text
host replication replicator 0.0.0.0/0 scram-sha-256
```

Save and exit:

* Press `ESC`
* Type:

```text
:wq
```

Exit container:

```bash
exit
```

---

# STEP 10 — Restart PRIMARY Container

```bash
docker restart pg-primary
```

Verify:

```bash
docker ps
```

---

# STEP 11 — Test Replication Login

```bash
docker exec -it pg-primary psql -U replicator -d postgres
```

Password:

```text
replpass
```

Exit:

```sql
\q
```

---

# STEP 12 — Create Base Backup Directory

Enter PRIMARY container:

```bash
docker exec -it pg-primary bash
```

Create standby backup folder:

```bash
mkdir /tmp/standby
```

---

# STEP 13 — Create Physical Base Backup

Run:

```bash
/usr/lib/postgresql/18/bin/pg_basebackup \
-h pg-primary \
-D /tmp/standby \
-U replicator \
-P \
-W \
-R
```

Password:

```text
replpass
```

Wait until:

```text
base backup completed
```

Exit container:

```bash
exit
```

---

# STEP 14 — Copy Backup To WSL Host

```bash
docker cp pg-primary:/tmp/standby ~/pg18-standby
```

Verify:

```bash
ls -ltr ~/pg18-standby
```

You should see:

* PG_VERSION
* base
* global
* pg_wal
* standby.signal

---

# STEP 15 — Start STANDBY Containers

## Standby 1

```bash
docker run -d \
--name pg-standby1 \
--network pgnet \
-p 6001:5432 \
-v ~/pg18-standby:/var/lib/postgresql/18/docker \
postgres:18
```

---

## Standby 2

```bash
docker run -d \
--name pg-standby2 \
--network pgnet \
-p 6002:5432 \
-v ~/pg18-standby:/var/lib/postgresql/18/docker \
postgres:18
```

---

## Standby 3

```bash
docker run -d \
--name pg-standby3 \
--network pgnet \
-p 6003:5432 \
-v ~/pg18-standby:/var/lib/postgresql/18/docker \
postgres:18
```

---

# STEP 16 — Verify Containers Running

```bash
docker ps
```

Expected:

```text
pg-primary
pg-standby1
pg-standby2
pg-standby3
```

All containers should be UP.

---

# STEP 17 — Verify Replication

Connect PRIMARY:

```bash
docker exec -it pg-primary psql -U postgres
```

Run:

```sql
SELECT client_addr, state
FROM pg_stat_replication;
```

Expected:

```text
streaming
```

Exit:

```sql
\q
```

---

# STEP 18 — Test Replication

Connect PRIMARY:

```bash
docker exec -it pg-primary psql -U postgres
```

Create test table:

```sql
CREATE TABLE test1(
    id INT,
    name TEXT
);

INSERT INTO test1 VALUES (1,'Clement');

SELECT * FROM test1;
```

Exit:

```sql
\q
```

---

# STEP 19 — Verify Data On STANDBY

Connect standby:

```bash
docker exec -it pg-standby1 psql -U postgres
```

Run:

```sql
SELECT * FROM test1;
```

Expected:

```text
1 | Clement
```

Exit:

```sql
\q
```

---

# STEP 20 — Verify STANDBY Is Read Only

Connect standby:

```bash
docker exec -it pg-standby1 psql -U postgres
```

Run:

```sql
INSERT INTO test1 VALUES (2,'TEST');
```

Expected:

```text
cannot execute INSERT in a read-only transaction
```

Exit:

```sql
\q
```

---

# STEP 21 — Manual Failover Test

Stop PRIMARY:

```bash
docker stop pg-primary
```

---

# STEP 22 — Promote STANDBY

Enter standby container:

```bash
docker exec -it pg-standby1 bash
```

Switch to postgres user:

```bash
su - postgres
```

Promote standby:

```bash
/usr/lib/postgresql/18/bin/pg_ctl promote \
-D /var/lib/postgresql/18/docker
```

Expected:

```text
server promoting
```

Exit:

```bash
exit
```

---

# STEP 23 — Verify STANDBY Became PRIMARY

Connect standby:

```bash
docker exec -it pg-standby1 psql -U postgres
```

Run:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

Meaning:

* standby promotion successful
* server is now PRIMARY

---

# STEP 24 — Test Writes After Failover

Run:

```sql
INSERT INTO test1 VALUES (2,'Failover Success');

SELECT * FROM test1;
```

Expected:

```text
1 | Clement
2 | Failover Success
```

Exit:

```sql
\q
```

---

# Useful Docker Commands

## View running containers

```bash
docker ps
```

---

## View all containers

```bash
docker ps -a
```

---

## View container logs

```bash
docker logs pg-primary
```

---

## Enter container shell

```bash
docker exec -it pg-primary bash
```

---

## Stop container

```bash
docker stop pg-primary
```

---

## Start container

```bash
docker start pg-primary
```

---

# Important PostgreSQL HA Concepts

| Concept               | Description                          |
| --------------------- | ------------------------------------ |
| WAL                   | Write Ahead Log used for replication |
| Streaming Replication | Real-time WAL shipping               |
| pg_basebackup         | Physical backup for standby creation |
| standby.signal        | Enables standby mode                 |
| pg_stat_replication   | Replication monitoring view          |
| pg_hba.conf           | Authentication configuration         |
| pg_ctl promote        | Promotes standby to primary          |
| pg_is_in_recovery()   | Identifies primary or standby        |

---

# Notes

* This is a learning PoC only.
* Multiple standbys sharing the same backup directory is not recommended for production.
* Production HA environments should use:

  * Patroni
  * repmgr
  * Kubernetes operators
  * Pgpool-II
  * HAProxy
  * Dedicated storage design

---

# Cleanup

```bash
docker stop pg-primary pg-standby1 pg-standby2 pg-standby3

docker rm pg-primary pg-standby1 pg-standby2 pg-standby3

docker volume rm pg_primary_data
```

---


```
```
