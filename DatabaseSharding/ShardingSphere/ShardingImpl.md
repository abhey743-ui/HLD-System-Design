# Spring Boot + ShardingSphere: Full Implementation & Configuration Guide

This document walks through the sharding project we built earlier — file by file,
config by config — and then covers the different sharding algorithms ShardingSphere
offers, so you know what other options exist beyond the simple `user_id % 2` example.

---

## Part 1: How the Implementation Works, End to End

### The big picture

Normally, a Spring Boot app talks to **one** database through a single `DataSource`.
With sharding, we replace that single `DataSource` with a "smart" one — provided by
ShardingSphere — that sits in front of **multiple physical databases**. Your
application code (entities, repositories, controllers) doesn't change; ShardingSphere
intercepts every query, works out which physical database it needs to hit, and routes
it there transparently.

```
Controller → Repository → JPA → ShardingSphereDriver → decides shard → real MySQL DB
```

### Project structure recap

```
sharding-demo/
├── pom.xml
├── sql/setup-shards.sql
├── src/main/resources/
│   ├── application.yml
│   └── sharding-config.yaml
└── src/main/java/com/example/shardingdemo/
    ├── ShardingDemoApplication.java
    ├── entity/Order.java
    ├── repository/OrderRepository.java
    └── controller/OrderController.java
```

---

## Part 2: Every Config File Explained

### `pom.xml` — Dependencies

The only dependency that's unusual compared to a normal Spring Boot project is:

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc</artifactId>
    <version>5.4.1</version>
</dependency>
```

This pulls in the ShardingSphere JDBC driver as a library — no separate server needed.
Everything else (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`,
`mysql-connector-j`) is completely standard Spring Boot boilerplate.

### `application.yml` — Pointing Spring at ShardingSphere

```yaml
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:sharding-config.yaml
```

This is the single most important change from a normal Spring Boot app. Instead of:

```yaml
url: jdbc:mysql://localhost:3306/mydb
```

...we point the datasource at the **ShardingSphereDriver**, and tell it where to find
the sharding rules (`sharding-config.yaml`, loaded from the classpath). From Spring's
point of view, this still looks like "just a datasource" — Spring doesn't know or care
that there's routing logic happening underneath.

### `sharding-config.yaml` — The Actual Sharding Rules

This file has three main sections:

**1. `dataSources` — the real, physical databases**

```yaml
dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/order_ds0?...
    username: root
    password: root
  ds1:
    ...
```

Each entry here is a genuine, physical MySQL connection, pooled through HikariCP.
`ds0` and `ds1` are just internal names — ShardingSphere refers to them by these
names elsewhere in the config.

**2. `rules` → `!SHARDING` → `tables` — how a logical table maps to physical shards**

```yaml
tables:
  t_order:
    actualDataNodes: ds$->{0..1}.t_order
    databaseStrategy:
      standard:
        shardingColumn: user_id
        shardingAlgorithmName: database_inline
    keyGenerateStrategy:
      column: order_id
      keyGeneratorName: snowflake
```

- `actualDataNodes: ds$->{0..1}.t_order` — this is ShardingSphere's shorthand
  meaning "the logical table `t_order` physically exists as a `t_order` table
  inside both `ds0` and `ds1`." The `$->{0..1}` is inline-expression syntax for
  "generate ds0 and ds1."
- `databaseStrategy.standard.shardingColumn: user_id` — tells ShardingSphere
  which column in the query to inspect to decide the shard.
- `shardingAlgorithmName: database_inline` — points to the algorithm defined
  further down that actually does the math.
- `keyGenerateStrategy` — since we removed `@GeneratedValue` from the entity
  (auto-increment IDs don't work safely across separate physical databases),
  ShardingSphere generates unique IDs itself using a Snowflake-style generator,
  guaranteeing no collisions between `ds0` and `ds1`.

**3. `shardingAlgorithms` — the actual routing logic**

```yaml
shardingAlgorithms:
  database_inline:
    type: INLINE
    props:
      algorithm-expression: ds$->{user_id % 2}
```

This is where the real decision happens. `INLINE` lets you write a small
Groovy-style expression: take `user_id`, compute `% 2`, and use the result to
pick `ds0` or `ds1`. So `user_id = 1001` → `1001 % 2 = 1` → routed to `ds1`.

**4. `props`**

```yaml
props:
  sql-show: true
```

Purely for visibility/debugging — logs the rewritten SQL and which shard it hit,
which is genuinely the best way to *see* sharding working while you're learning it.

### `sql/setup-shards.sql` — Creating the physical shards

ShardingSphere doesn't create your physical databases/tables for you in this basic
setup — you run this script once against MySQL to create `order_ds0` and `order_ds1`,
each with its own real `t_order` table, matching what `actualDataNodes` expects to find.

### `Order.java` (Entity)

Maps to the **logical** `t_order` table. Notice there's no `@GeneratedValue` — id
generation is handled by ShardingSphere's key generator instead, since a database's
own auto-increment can't safely coordinate across two separate physical databases.

### `OrderRepository.java`

A completely ordinary `JpaRepository`. This is deliberate — there is *zero*
sharding-specific code here. `findByUserId(...)` works exactly like it would
against a single database; ShardingSphere is what notices `user_id` in the
generated SQL and uses it to route the query correctly.

### `OrderController.java`

Three endpoints show the three access patterns:
- `POST /orders` — write, routed by `user_id`.
- `GET /orders/user/{userId}` — read, routed to a single shard (fast).
- `GET /orders` — read with **no** shard key present, so ShardingSphere has to
  broadcast the query to **every** shard and merge the results — noticeably
  more expensive, and a good illustration of the main tradeoff of sharding.

---

## Part 3: Sharding Algorithms — What Else Is Available

`user_id % 2` (an `INLINE` algorithm) is just one option. ShardingSphere ships
several built-in algorithms, grouped into three categories: **Auto Sharding**,
**Standard Sharding**, and **Complex/Hint Sharding**. Here's what each one does,
and when you'd reach for it.

### Auto Sharding Algorithms
*("Syntactic sugar" — you don't manage the shard topology yourself; ShardingSphere handles table creation and distribution automatically.)*

**`MOD`** — Modulo Sharding
- Takes the shard key's numeric value and computes `value % shardingCount`.
- Simple, predictable, evenly distributes data **if** your key values are
  reasonably spread out (e.g. random or sequential IDs).
- Weak point: if your key values cluster (e.g. mostly even numbers), distribution
  can skew.

**`HASH_MOD`** — Hash Modulo Sharding
- Hashes the shard key first, *then* applies modulo on the hash.
- Fixes `MOD`'s weak point — even if your raw key values aren't well distributed,
  hashing scrambles them first, giving a much more even spread across shards.
- This is usually the safer default over plain `MOD` for arbitrary keys like UUIDs
  or user IDs that might not be perfectly sequential.

**`VOLUME_RANGE`** — Volume-Based Range Sharding
- You define a `range-lower`, `range-upper`, and `sharding-volume` (chunk size).
  Data is split into equal-sized ranges — e.g. IDs 1–1,000,000 → shard 0,
  1,000,001–2,000,000 → shard 1, and so on.
- Good when you want range-based sharding but don't want to hand-pick each
  boundary — ShardingSphere calculates the ranges for you based on volume.

**`BOUNDARY_RANGE`** — Boundary-Based Range Sharding
- Similar to volume-based, but instead of equal-sized chunks, *you* specify the
  exact boundary values (e.g. "shard 0 = below 500,000, shard 1 = 500,000 to
  2,000,000, shard 2 = above that").
- Useful when your data isn't evenly distributed over time/value and you want
  uneven, deliberately-chosen ranges instead.

**`AUTO_INTERVAL`** — Mutable Interval Sharding (time-based)
- Splits data by time intervals (e.g. one shard per month), and — unlike the
  fixed version below — can automatically create new shards/tables as time moves
  forward, so you don't run out of buckets.
- Common for logs, orders, or time-series-style data where "this month's data"
  and "last month's data" naturally belong in different places.

### Standard Sharding Algorithms
*(You write the exact expression yourself — more control, more manual work.)*

**`INLINE`** — Inline Expression Sharding *(what we used)*
- You write a short Groovy-like expression, e.g. `ds$->{user_id % 2}` or
  `t_order_$->{order_id % 4}`.
- Supports `=` and `IN` conditions well. Supports range queries (`BETWEEN`, `>`,
  `<`) only if you explicitly enable `allow-range-query-with-inline-sharding` —
  and even then, range queries have to broadcast to all shards since the
  expression can't "reverse" a range into specific shard numbers.
- Best for: full control over the exact formula, when your logic is simple
  enough to express in one line (which covers most real cases).

**`INTERVAL`** — Fixed Interval Sharding
- Like `AUTO_INTERVAL`, but you predefine the exact set of shards/time-ranges up
  front rather than letting ShardingSphere expand them automatically.
- Use when you know your full time range in advance (e.g. archiving exactly 12
  months of historical data) and don't need auto-expansion.

**`CLASS_BASED`** — Custom Java Class Sharding
- You write your own Java class implementing ShardingSphere's algorithm interface,
  giving you complete, arbitrary logic (e.g. "route based on a customer's
  subscription tier stored in a lookup table," or any business rule too complex
  for a one-line expression).
- Most flexible option, but also the most work — you own the correctness and
  performance of that logic entirely.

### Complex & Hint Sharding Algorithms
*(For sharding decisions based on more than one column, or based on context outside the SQL itself.)*

**`COMPLEX_INLINE`** — Complex Inline Sharding
- Like `INLINE`, but supports **multiple** sharding columns at once — e.g.
  routing based on a combination of `user_id` AND `region_code` together, rather
  than just one column.
- Useful when a single column isn't a good enough shard key alone (e.g. two
  users could have the same numeric ID pattern but need to land in different
  shards depending on which region they're in too).

**`HINT_INLINE`** — Hint-Based Inline Sharding
- Routes based on a "hint" you set manually in your application code at query
  time, rather than inferring it from a column in the SQL itself.
- Useful for edge cases — e.g. an admin/reporting query that needs to explicitly
  target one specific shard, or scenarios where the shard key genuinely isn't
  present anywhere in the query.

---

## Part 4: Quick Decision Guide

| Your situation | Algorithm to consider |
|---|---|
| Simple, single numeric/string key, you want full control | `INLINE` |
| Same as above, but key values aren't well distributed | `HASH_MOD` |
| Data naturally splits by time (logs, orders, events) | `AUTO_INTERVAL` or `INTERVAL` |
| You want ShardingSphere to auto-manage table creation | Any "Auto Sharding" type (`MOD`, `HASH_MOD`, `VOLUME_RANGE`, etc.) |
| Sharding decision needs more than one column | `COMPLEX_INLINE` |
| Sharding logic is too complex for an expression (business rules, lookups) | `CLASS_BASED` |
| Shard key isn't present in the query at all | `HINT_INLINE` |

---

## Summary

The implementation we built uses the simplest, most common combination: **standard
sharding strategy + `INLINE` algorithm + a single shard key (`user_id`)**. This
covers the overwhelming majority of real-world use cases. The other algorithms
above exist for specific situations — uneven key distribution, time-based data,
multi-column routing, or business logic too complex to express in one line — and
you'd reach for them only once your actual data or query patterns demand it.
