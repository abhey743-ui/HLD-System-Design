# Spring Boot + ShardingSphere: Read-Write Splitting Implementation & Configuration Guide

This document walks through the read-write splitting project — file by file,
config by config — and covers how it differs from the sharding setup we built
earlier, plus the load-balancing options and the one real gotcha (replication lag)
you need to design around.

---

## Part 1: What Read-Write Splitting Actually Does

Recap: unlike sharding (where different physical databases hold **different**
data), read-write splitting uses databases that all hold **the same** data —
one **primary** that accepts writes, and one or more **replicas** that are kept
in sync with the primary through native database replication, and that handle
**read** traffic only.

```
       WRITE
        ↓
   [ primary ]  --MySQL replication-->  [ replica0 ]
                --MySQL replication-->  [ replica1 ]
       READ ↑ (round-robin between replicas)
```

The goal isn't to split up storage — it's to take read traffic (usually the
vast majority of traffic in most apps) off the primary, so the primary can
focus on writes without being slowed down by read queries competing for its
resources.

### Project structure recap

```
readwrite-split-demo/
├── pom.xml
├── docker-compose.yml
├── sql/setup-primary.sql
├── src/main/resources/
│   ├── application.yml
│   └── readwrite-config.yaml
└── src/main/java/com/example/rwdemo/
    ├── ReadWriteSplitDemoApplication.java
    ├── entity/Order.java
    ├── repository/OrderRepository.java
    └── controller/OrderController.java
```

---

## Part 2: Every Config File Explained

### `docker-compose.yml` — Setting up the primary + replicas

This is new compared to the sharding project. Since read-write splitting relies
on the databases already being kept in sync via **native database replication**
(not application code, not ShardingSphere), we need actual MySQL replication
configured between three instances. The compose file uses Bitnami's MySQL image,
which handles the replication setup through environment variables:

```yaml
mysql-primary:
  environment:
    - MYSQL_REPLICATION_MODE=master
    ...
mysql-replica0:
  environment:
    - MYSQL_REPLICATION_MODE=slave
    - MYSQL_MASTER_HOST=mysql-primary
    ...
```

This spins up one primary on port 3306 and two replicas on 3307/3308, with
replication already wired up between them. **ShardingSphere plays no role in
keeping the data in sync** — that's MySQL's own job. ShardingSphere's only job
is deciding *which* of these three databases a given query should go to.

### `application.yml` — Pointing Spring at ShardingSphere

```yaml
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:readwrite-config.yaml
```

Identical pattern to the sharding project — Spring's datasource points at the
ShardingSphere driver, which loads its rules from `readwrite-config.yaml`.

### `readwrite-config.yaml` — The Read-Write Splitting Rule

**1. `dataSources` — the three real, physical databases**

```yaml
dataSources:
  primary:
    jdbcUrl: jdbc:mysql://localhost:3306/orders_primary?...
  replica0:
    jdbcUrl: jdbc:mysql://localhost:3307/orders_primary?...
  replica1:
    jdbcUrl: jdbc:mysql://localhost:3308/orders_primary?...
```

Three separate physical connections — same as before, just three real MySQL
instances (ports 3306, 3307, 3308) instead of two.

**2. `rules` → `!READWRITE_SPLITTING` — the routing rule**

```yaml
rules:
  - !READWRITE_SPLITTING
    dataSources:
      readwrite_ds:
        staticStrategy:
          writeDataSourceName: primary
          readDataSourceNames:
            - replica0
            - replica1
        loadBalancerName: round-robin-lb
```

- `readwrite_ds` is a **logical** data source name. Your app never references
  `primary`, `replica0`, or `replica1` directly — it just talks to
  `readwrite_ds`, and ShardingSphere decides, per statement, which physical
  database actually receives it.
- `staticStrategy` means the primary/replica assignment is fixed in config
  (there's also a `dynamicStrategy`, covered below, for auto-discovery).
- `writeDataSourceName: primary` — every INSERT/UPDATE/DELETE goes here.
- `readDataSourceNames` — the pool of databases eligible to serve SELECT
  queries.
- `loadBalancerName` — points to the algorithm deciding *which* replica
  handles each individual read.

**3. `loadBalancers` — how reads get distributed across replicas**

```yaml
loadBalancers:
  round-robin-lb:
    type: ROUND_ROBIN
```

`ROUND_ROBIN` alternates requests evenly: read 1 → replica0, read 2 → replica1,
read 3 → replica0, and so on. (More load balancer types below.)

**4. `props`**

```yaml
props:
  sql-show: true
```

Same as before — logs which physical database actually served each statement,
so you can literally watch writes go to the primary and reads bounce between
replicas.

### `sql/setup-primary.sql` — Creating the schema

Notice this only runs against the **primary** (port 3306). Because replication
is already active, MySQL propagates the table structure — and every row
inserted afterward — to both replicas automatically. You never manually run
setup scripts against the replicas.

### `Order.java`, `OrderRepository.java`

Both completely ordinary — same pattern as the sharding demo. No routing logic
anywhere in the entity or repository; ShardingSphere handles it transparently
based on whether the generated SQL is a read or a write.

### `OrderController.java`

Four endpoints demonstrate the behavior:
- `POST /orders` — a write, always routed to `primary`.
- `GET /orders`, `GET /orders/user/{userId}` — reads, routed to whichever
  replica the round-robin picks next.
- `POST /orders/create-and-read-back` — wraps a write and an immediate read in
  one `@Transactional` block. ShardingSphere routes **everything inside one
  transaction to the primary**, which is the standard fix for the replication
  lag problem explained below.

---

## Part 3: Load Balancer Options for Reads

`ROUND_ROBIN` is the simplest choice, but ShardingSphere supports a couple of
others worth knowing about:

| Type | Behavior | When to use |
|---|---|---|
| `ROUND_ROBIN` | Alternates evenly across replicas in fixed order | Default choice — replicas are roughly equal in capacity |
| `RANDOM` | Picks a replica at random for each read | Similar effect to round robin, simpler internally, fine for most cases |
| `WEIGHT` | You assign a weight per replica (e.g. a beefier replica gets more traffic) | Replicas have different hardware/capacity and shouldn't get equal load |

For most setups where all replicas are provisioned identically, `ROUND_ROBIN`
is the standard, simplest pick — it's what we used.

---

## Part 4: Static vs Dynamic Strategy

The config above uses `staticStrategy` — you hardcode which data source is the
primary and which are replicas. ShardingSphere also supports `dynamicStrategy`,
which integrates with a **database discovery service** to automatically detect
which node is currently the primary (useful in setups with automatic failover,
where the primary can change after a failure without you manually updating
config). For a straightforward setup like this demo, `staticStrategy` is
simpler and perfectly adequate; `dynamicStrategy` becomes worth the extra setup
once you have automated primary failover in place at the infrastructure level.

---

## Part 5: The One Real Gotcha — Replication Lag

Because replicas sync from the primary **asynchronously**, there's always a
small window where a replica hasn't yet received the latest write — usually
milliseconds, but it can stretch longer under heavy write load or network
issues. If your app writes something and then immediately reads it back
through a plain, un-wrapped query, there's a chance it lands on a replica that
hasn't caught up yet, and the read appears to "miss" data that was just written.

**The standard fix:** wrap the write and the follow-up read in the same
database transaction (`@Transactional` in Spring). ShardingSphere detects that
both statements belong to one transaction and routes the whole thing to the
primary, guaranteeing the read sees the fresh data. This is exactly what the
`create-and-read-back` endpoint in the controller demonstrates.

**Important:** don't apply this fix everywhere by default. If you wrap every
read in a transaction that forces it to the primary, you've quietly defeated
the entire purpose of read-write splitting — you're back to hammering the
primary with all your traffic. Use it selectively, only for the specific flows
where "the user must see their own write immediately" genuinely matters (like
right after a form submission), and leave everything else as plain, replica-served reads.

---

## Part 6: Read-Write Splitting vs Sharding — Quick Comparison

| | Sharding | Read-Write Splitting |
|---|---|---|
| Physical databases hold | Different data | Identical data (mirrors) |
| Solves | Storage/write-volume limits on one server | Read-traffic volume on one server |
| Sync mechanism | None needed (data isn't duplicated) | Native database replication |
| Routing decided by | A column's value | Whether the statement is a read or write |
| Can combine both? | Yes — shard for write scale, then read-write-split each shard for read scale |

That last row is worth calling out: you're not limited to picking one. A
common real-world pattern is to shard your data across multiple primaries
for write scaling, and give *each* shard's primary its own set of read
replicas — combining both techniques for a system that scales on both axes.

---

## Summary

Read-write splitting is a much lighter change than sharding — no data gets
split up, no shard key needs choosing, and the sync between databases is
handled entirely by the database engine itself rather than by application-level
sync code. The main configuration decisions are just: which database is the
primary, which are replicas, and which load-balancing strategy spreads read
traffic across them. The only real design care needed is around replication
lag, and that's solved narrowly — with transactions — rather than by avoiding
read-write splitting altogether.
