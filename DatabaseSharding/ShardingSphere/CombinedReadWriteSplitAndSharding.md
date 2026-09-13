# Combining Data Sharding + Read-Write Splitting — Complete Guide

This document explains **why** you'd combine sharding and read-write splitting,
**what each one alone cannot solve**, and walks through a **complete,
zero-skipped YAML configuration** in current ShardingSphere 5.x syntax, followed
by a step-by-step trace of exactly what happens to a request.

---

## Part 1: What Are We Actually Trying to Achieve?

Imagine an e-commerce system with a `t_order` table that has grown into a real
problem:

- It has **hundreds of millions of rows** — too much for one database to store
  and index efficiently.
- It also gets **hammered with read traffic** — way more `SELECT`s than
  `INSERT`s, since customers check their orders far more often than they place
  new ones.

These are **two separate bottlenecks**, and each needs a different fix.

### What sharding alone cannot solve

Sharding splits your data across multiple databases (say, `ds0` and `ds1`),
which fixes the storage/write-volume problem — no single database has to hold
all the data. **But** each shard (`ds0`, `ds1`) is still just one database
server. If read traffic against `ds0` is heavy, sharding doesn't help — `ds0`
alone still has to handle every read that lands on it. Sharding spreads data
out; it does nothing to spread out the *load on a single shard*.

### What read-write splitting alone cannot solve

Read-write splitting takes one database, gives it a few replicas, and spreads
read traffic across them. This fixes the read-load problem beautifully —
**but only for a single database.** If your data has grown too large to fit
on one primary + its replicas in the first place, read-write splitting alone
doesn't help — you still have one giant primary that can't hold or write all
the data fast enough.

### Why combining them solves both

If you **shard first**, you get multiple smaller, independent database groups
(`ds0`, `ds1`), each responsible for a portion of the data — this solves the
storage/write problem. Then, if you give **each shard its own primary + replicas**,
you solve the read-load problem *within* every shard too. The two techniques
operate on different axes and stack cleanly:

```
                     SHARDING splits data
                    ↙                    ↘
              Shard 0 (subset of data)     Shard 1 (subset of data)
                    ↓                            ↓
       READ-WRITE SPLITTING              READ-WRITE SPLITTING
       spreads read load                 spreads read load
                    ↓                            ↓
    [master] [replica] [replica]      [master] [replica] [replica]
```

**In one sentence:** sharding decides *which group of servers* your data lives
on; read-write splitting decides *which specific server within that group*
handles a given query. They're not competing solutions — they answer two
completely different questions, and a large system usually needs both.

---

## Part 2: The Complete, Combined Configuration (ShardingSphere 5.x)

Below is a full config for two shards (`ds0`, `ds1`), each with a master and
two replicas, sharding two related tables (`t_order`, `t_order_item`) by
`user_id` at the database level and `order_id` at the table level, plus one
broadcast table (`t_config`). Nothing is skipped or abbreviated.

```yaml
# =========================================================================
# 1. PHYSICAL DATA SOURCES
#    Six real, physical MySQL connections in total:
#    ds0 (master) + ds0_slave0 + ds0_slave1
#    ds1 (master) + ds1_slave0 + ds1_slave1
# =========================================================================
dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/ds0?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8
    username: root
    password: root
    maximumPoolSize: 10

  ds0_slave0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3307/ds0?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8
    username: root
    password: root
    maximumPoolSize: 10

  ds0_slave1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3308/ds0?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8
    username: root
    password: root
    maximumPoolSize: 10

  ds1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3309/ds1?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8
    username: root
    password: root
    maximumPoolSize: 10

  ds1_slave0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3310/ds1?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8
    username: root
    password: root
    maximumPoolSize: 10

  ds1_slave1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3311/ds1?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8
    username: root
    password: root
    maximumPoolSize: 10

# =========================================================================
# 2. RULES — two rules stacked: READWRITE_SPLITTING first, SHARDING second
# =========================================================================
rules:

  # -----------------------------------------------------------------------
  # RULE 1: READWRITE_SPLITTING
  # Groups the six physical data sources above into two LOGICAL data
  # sources: "readwrite_ds_0" and "readwrite_ds_1". Each logical name
  # bundles one master + its replicas, with a load balancer deciding
  # which replica serves any given read.
  # -----------------------------------------------------------------------
  - !READWRITE_SPLITTING
    dataSources:
      readwrite_ds_0:
        staticStrategy:
          writeDataSourceName: ds0
          readDataSourceNames:
            - ds0_slave0
            - ds0_slave1
        loadBalancerName: round-robin-lb

      readwrite_ds_1:
        staticStrategy:
          writeDataSourceName: ds1
          readDataSourceNames:
            - ds1_slave0
            - ds1_slave1
        loadBalancerName: round-robin-lb

    loadBalancers:
      round-robin-lb:
        type: ROUND_ROBIN

  # -----------------------------------------------------------------------
  # RULE 2: SHARDING
  # This is the important part: the sharding rule's actualDataNodes
  # reference "readwrite_ds_0" / "readwrite_ds_1" — the LOGICAL names from
  # the rule above — not raw physical databases. Sharding sits "on top of"
  # read-write splitting: it first decides which logical group a row
  # belongs to, and read-write splitting then decides which physical
  # server within that group actually serves the query.
  # -----------------------------------------------------------------------
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: readwrite_ds_${0..1}.t_order_${0..1}
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: database_inline
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: order_table_inline
        keyGenerateStrategy:
          column: order_id
          keyGeneratorName: snowflake

      t_order_item:
        actualDataNodes: readwrite_ds_${0..1}.t_order_item_${0..1}
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: database_inline
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: order_item_table_inline
        keyGenerateStrategy:
          column: order_item_id
          keyGeneratorName: snowflake

    # t_order and t_order_item are always sharded together the same way
    # (both use user_id for the database, order_id for the table), so we
    # declare them as "binding tables." This tells ShardingSphere it can
    # join them without cross-shard scatter-gather, since a given order
    # and its items always live in the same physical location.
    bindingTables:
      - t_order,t_order_item

    # t_config is small, rarely-changing reference data (e.g. status codes,
    # config flags) that every shard needs a full copy of. Broadcast tables
    # get written to ALL shards on every write, and any shard can answer a
    # read for them.
    broadcastTables:
      - t_config

    shardingAlgorithms:
      database_inline:
        type: INLINE
        props:
          algorithm-expression: readwrite_ds_${user_id % 2}

      order_table_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_${order_id % 2}

      order_item_table_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_item_${order_id % 2}

    keyGenerators:
      snowflake:
        type: SNOWFLAKE

# =========================================================================
# 3. PROPS — global behavior settings
# =========================================================================
props:
  sql-show: true   # logs the rewritten SQL AND which physical data source it hit — essential while learning/debugging
```

### `application.yml` (unchanged pattern)

```yaml
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:combined-config.yaml
  jpa:
    hibernate:
      ddl-auto: none
    show-sql: true
```

Same as every example so far — Spring just points at ShardingSphere, and
ShardingSphere loads the combined config above.

---

## Part 3: Config Section-by-Section Recap

| Section | What it defines |
|---|---|
| `dataSources` | The 6 real, physical MySQL connections |
| `!READWRITE_SPLITTING` rule | Groups physical DBs into 2 logical master+replica groups |
| `!SHARDING` rule → `tables` | Which logical group + which table a row belongs to |
| `bindingTables` | Related tables sharded identically, enabling shard-local joins |
| `broadcastTables` | Small reference tables copied to every shard |
| `shardingAlgorithms` | The actual `% 2` math deciding database and table |
| `keyGenerators` | Snowflake ID generation so IDs never collide across shards |
| `props` | Debug/behavior flags like `sql-show` |

---

## Part 4: Tracing a Request Through the Whole System

### Scenario A — Writing a new order

**Request:** create an order for `user_id = 1001`.

1. **Sharding decides the group.** `user_id % 2` → `1001 % 2 = 1` → this row
   belongs to **`readwrite_ds_1`**.
2. **Sharding decides the table.** The generated `order_id` (via Snowflake,
   say it comes out as `88`) → `88 % 2 = 0` → the row goes into
   **`t_order_0`**.
3. **Read-write splitting decides the physical server.** Since this is an
   `INSERT`, and the logical group is `readwrite_ds_1`, the write is sent to
   **`ds1`** (the master) — never a replica.
4. MySQL's own replication then copies this new row from `ds1` to
   `ds1_slave0` and `ds1_slave1` in the background.
5. **Result:** the row physically lands in `ds1.t_order_0`, and shortly after,
   identical copies exist in `ds1_slave0.t_order_0` and `ds1_slave1.t_order_0`.

### Scenario B — Reading that same user's orders

**Request:** `findByUserId(1001)`.

1. **Sharding decides the group.** Same math as before: `1001 % 2 = 1` →
   **`readwrite_ds_1`**. (No table-level shard key is present in this query,
   so it will check both `t_order_0` and `t_order_1` within that group.)
2. **Read-write splitting decides the physical server.** Since this is a
   `SELECT`, and we're targeting `readwrite_ds_1`, the query is sent to
   whichever replica round-robin picks next — say **`ds1_slave0`** this time,
   **`ds1_slave1`** next time.
3. **Result:** the read never touches the master `ds1` at all, and it never
   touches anything in `readwrite_ds_0`/`ds0` either, since the shard key
   ruled that whole group out entirely.

### Scenario C — A query with no shard key at all (e.g. an admin "all orders" report)

1. **Sharding** has nothing to narrow down with, so it broadcasts the query to
   **both** `readwrite_ds_0` and `readwrite_ds_1`.
2. **Read-write splitting** still applies within each: since it's a `SELECT`,
   each group serves it from one of its own replicas (not the masters).
3. ShardingSphere merges the results from all four replicas involved
   (`ds0_slave*` and `ds1_slave*`) into a single result set handed back to
   your application.
4. **This is the expensive case** — it's why unscoped, shard-key-less queries
   should be used sparingly; they lose the main performance benefit sharding
   was supposed to provide.

### Scenario D — Writing an order and immediately reading it back (same transaction)

1. Wrapping both statements in one `@Transactional` block causes ShardingSphere
   to route the whole transaction to the **master** of whichever shard group
   the `user_id` resolves to — bypassing replicas entirely for the duration
   of that transaction.
2. This sidesteps replication lag: the read is guaranteed to see the write
   that just happened, because both went to the same physical database.

---

## Summary

- **Sharding** answers: *which group of database servers does this data belong to?*
- **Read-write splitting** answers: *which specific server within that group should serve this particular query?*
- Combined, a single logical table like `t_order` can be spread across multiple
  shards for write/storage scale, while every shard independently spreads its
  own read load across replicas — solving both bottlenecks at once, using one
  unified YAML configuration where the sharding rule's data nodes simply point
  at the logical names the read-write-splitting rule defines.
