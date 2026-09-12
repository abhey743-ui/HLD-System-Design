# ShardingSphere Data Encryption (Encrypt Rule) — Complete Guide

This is the last major piece of the puzzle alongside sharding and read-write
splitting: **data encryption** (sometimes called "data masking"). This
document explains what it is, how it works conceptually, breaks down the
legacy config you found, converts it to current ShardingSphere 5.x syntax,
and traces exactly what happens on disk and over the wire for both a write
and a read.

---

## Part 1: What Is This, and Why Does It Exist?

**The problem:** you have sensitive columns — a user's phone number, ID
number, salary, or similar — and you don't want that data sitting in your
database as plain, readable text. If the database is ever breached, leaked
in a backup, or viewed by someone with raw DB access who shouldn't see it,
plaintext sensitive data is a direct liability.

**The naive fix** would be to encrypt/decrypt the values yourself in your
application code before every save and after every read. This works, but it
means every single query that touches that column needs custom
encryption-aware code, and it's easy to forget a spot.

**What ShardingSphere's Encrypt feature does instead:** it sits at the same
layer as sharding and read-write splitting — intercepting your SQL before it
reaches the database — and transparently:
- **Encrypts values on the way in** (right before an `INSERT`/`UPDATE` is sent
  to the real database).
- **Decrypts values on the way out** (right after a `SELECT` result comes
  back, before your application ever sees it).

Your application code — your entity, your repository, your controller —
**never encrypts or decrypts anything itself.** You write `user_id = 1001`
in your Java code as if it were plain data; ShardingSphere handles turning
that into ciphertext for storage and back into plaintext for you to read.

**The core mechanic: a "logical column" that maps to real physical column(s).**
You never store data under the column name your app uses. Instead, ShardingSphere
maintains a mapping: your app writes/reads a column called (for example)
`user_id`, but on disk, the actual table might have a column called
`user_cipher` holding the encrypted value. ShardingSphere quietly translates
between the two on every query.

---

## Part 2: Breaking Down the Config You Found

Here's what you posted, annotated line by line. This is the **legacy
ShardingSphere 4.x syntax**:

```yaml
dataSource:  !!org.apache.commons.dbcp2.BasicDataSource
  driverClassName: com.mysql.jdbc.Driver
  url: jdbc:mysql://127.0.0.1:3306/encrypt?serverTimezone=UTC&useSSL=false
  username: root
  password:

encryptRule:
  encryptors:
    encryptor_aes:
      type: aes
      props:
        aes.key.value: 123456abc
    encryptor_md5:
      type: md5

  tables:
    t_encrypt:
      columns:
        user_id:
          plainColumn: user_plain
          cipherColumn: user_cipher
          encryptor: encryptor_aes
        order_id:
          cipherColumn: order_cipher
          encryptor: encryptor_md5

props:
  query.with.cipher.column: true
```

**Piece by piece:**

- **`dataSource`** — the single real, physical database connection. (Note:
  encryption doesn't require multiple databases like sharding or read-write
  splitting do — it can be layered onto just one database, or combined with
  the others.)

- **`encryptRule.encryptors`** — defines the actual algorithms available to
  use:
  - `encryptor_aes` — type `aes`, a **reversible** algorithm. AES can encrypt
    *and* decrypt, which is why it needs a secret key (`aes.key.value`) —
    that key is what allows turning ciphertext back into the original value.
  - `encryptor_md5` — type `md5`, a **one-way hash**. MD5 cannot be reversed
    back into the original value — it's not really "encryption" in the sense
    of being recoverable, it's a fingerprint. You'd use this for values you
    only ever need to *compare* (like checking if a password matches), never
    values you need to read back in original form.

- **`encryptRule.tables.t_encrypt.columns`** — this is the actual mapping,
  defined per logical column:
  - **`user_id`** column:
    - `cipherColumn: user_cipher` — the real physical column storing the
      AES-encrypted value.
    - `plainColumn: user_plain` — an *additional* real physical column that
      keeps the original, unencrypted value alongside the encrypted one.
      This existed in 4.x mainly to support **gradual migration** — you could
      run with both plain and cipher columns temporarily while migrating an
      existing plaintext system to encryption, then drop the plain column
      once you're confident everything works.
    - `encryptor: encryptor_aes` — which algorithm above handles this column.
  - **`order_id`** column:
    - `cipherColumn: order_cipher`, `encryptor: encryptor_md5` — no
      `plainColumn` here, meaning this column has **no** original-value
      fallback stored at all; it's purely a one-way hash, presumably used
      only for exact-match lookups, not for reading the original order ID
      back out.

- **`props: query.with.cipher.column: true`** — a global toggle: when
  `true`, reads pull from the **cipher** column and decrypt on the way out.
  If it were `false` (and a `plainColumn` exists), reads could instead just
  pull straight from the plaintext column, skipping decryption — useful
  during a migration window before you've fully committed to encryption-only
  reads.

---

## Part 3: The Same Config in Current ShardingSphere 5.x Syntax

The legacy `plainColumn` / `queryWithCipherColumn` options were **removed** in
newer ShardingSphere versions (5.4.0+) to simplify the feature — production
encryption setups are expected to commit to cipher-only storage rather than
keep a plaintext fallback column around indefinitely. The current syntax also
groups related columns (cipher, assisted-query, like-query) into clearer
nested blocks instead of flat keys.

```yaml
dataSources:
  encrypt_ds:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/encrypt?serverTimezone=UTC&useSSL=false&characterEncoding=UTF-8
    username: root
    password: root

rules:
  - !ENCRYPT
    tables:
      t_encrypt:
        columns:
          user_id:
            cipher:
              name: user_cipher
              encryptorName: encryptor_aes
          order_id:
            cipher:
              name: order_cipher
              encryptorName: encryptor_md5

    encryptors:
      encryptor_aes:
        type: AES
        props:
          aes-key-value: 123456abc
      encryptor_md5:
        type: MD5

props:
  sql-show: true
```

**What changed, and why:**

- `dataSource` (singular, custom object tag) → `dataSources` (a proper map,
  matching the same style used everywhere else in modern ShardingSphere
  config — sharding, read-write splitting, and encryption now all share one
  consistent config shape).
- `encryptRule:` (a flat top-level key) → the encryption rule is now just one
  entry inside the shared `rules:` list, tagged `!ENCRYPT` — exactly the same
  pattern as `!SHARDING` and `!READWRITE_SPLITTING`, which is what lets you
  combine all three cleanly in one file.
- `cipherColumn: user_cipher` (flat key) → `cipher: { name: user_cipher,
  encryptorName: ... }` (nested block) — grouping the column name and its
  algorithm together, and leaving room for equivalent `assistedQuery:` and
  `likeQuery:` blocks (covered below) using the same shape.
- `plainColumn` and `query.with.cipher.column` → **removed entirely.** Modern
  ShardingSphere expects cipher-only storage; there's no built-in "keep a
  plaintext copy" option anymore.

---

## Part 4: Two More Column Types You'll See in Real Configs

Beyond the plain cipher column, ShardingSphere supports two additional,
optional column types per encrypted field — worth knowing since real-world
encrypt configs often include them:

**`assistedQuery`** — an extra column storing a *deterministic* transformation
of the value (often a keyed hash), used purely so you can still run exact-match
`WHERE` lookups efficiently. AES ciphertext for the same input can differ
between encryptions (depending on the mode used), so a raw `WHERE user_cipher =
?` might not reliably match. The assisted-query column exists specifically to
make equality lookups fast and correct even though the main cipher column
isn't reliably comparable.

**`likeQuery`** — a separate column enabling fuzzy `LIKE '%...%'` searches on
encrypted text, which is otherwise impossible since encrypting a whole string
destroys any partial-match structure. ShardingSphere applies a special
fuzzy-search-friendly algorithm (e.g. `CHAR_DIGEST_LIKE`) to populate this
column, letting you run limited fuzzy searches without ever storing the
value in plaintext.

Example showing all three together on one column:

```yaml
rules:
  - !ENCRYPT
    tables:
      t_user:
        columns:
          username:
            cipher:
              name: username_cipher
              encryptorName: aes_encryptor
            assistedQuery:
              name: username_assisted
              encryptorName: assisted_encryptor
            likeQuery:
              name: username_like
              encryptorName: like_encryptor
    encryptors:
      aes_encryptor:
        type: AES
        props:
          aes-key-value: 123456abc
      assisted_encryptor:
        type: MD5
      like_encryptor:
        type: CHAR_DIGEST_LIKE
```

This means one logical `username` column can actually be backed by **three**
real physical columns under the hood: one for storage (`username_cipher`),
one enabling exact lookups (`username_assisted`), and one enabling fuzzy
search (`username_like`) — and your application only ever refers to
`username`.

---

## Part 5: Tracing a Request Through the Encryption Layer

### Scenario A — Inserting a new row

**Your code writes (in plain Java/SQL terms):**
```sql
INSERT INTO t_encrypt (user_id, order_id) VALUES (1001, 55);
```

1. ShardingSphere intercepts this before it reaches MySQL.
2. It sees `user_id` is an encrypted logical column mapped to `encryptor_aes`.
   It runs `1001` through the AES algorithm using the configured key,
   producing ciphertext — say, `xK9$mQ...` (illustrative).
3. It sees `order_id` is mapped to `encryptor_md5`. It runs `55` through MD5,
   producing a hash — say, `7f4a8e...`.
4. It **rewrites the actual SQL** sent to MySQL to reference the real
   physical columns:
   ```sql
   INSERT INTO t_encrypt (user_cipher, order_cipher) VALUES ('xK9$mQ...', '7f4a8e...');
   ```
5. **What actually lands on disk:** the physical `t_encrypt` table has columns
   `user_cipher` and `order_cipher` — no `user_id` or `order_id` column exists
   at all in the real schema. Anyone querying the raw database directly sees
   only ciphertext/hashes, never the real values.

### Scenario B — Reading it back

**Your code queries:**
```sql
SELECT user_id FROM t_encrypt WHERE order_id = 55;
```

1. ShardingSphere rewrites the `WHERE` clause first: since `order_id` maps to
   `encryptor_md5` (one-way), it hashes `55` into `7f4a8e...` and rewrites the
   query to filter on the real column:
   ```sql
   SELECT user_cipher FROM t_encrypt WHERE order_cipher = '7f4a8e...';
   ```
2. MySQL executes this and returns the raw row containing `user_cipher =
   'xK9$mQ...'`.
3. ShardingSphere intercepts the result **before** handing it back to your
   application. It sees the result came from a column mapped to
   `encryptor_aes`, decrypts `'xK9$mQ...'` back into `1001` using the same
   key.
4. Your application receives `user_id = 1001` — a plain, readable integer —
   with zero awareness that any encryption/decryption happened.

### Scenario C — Why the `order_id = 55` lookup even worked

This is worth calling out explicitly: `order_id` uses **MD5**, a one-way
hash. ShardingSphere didn't "decrypt" `55` to compare it — it took your
query's literal value (`55`), hashed it the *same* way it was hashed at
insert time, and compared hash-to-hash. This works perfectly for exact-match
lookups but is exactly why you can never run `SELECT order_id FROM
t_encrypt` and get a real order ID back out — MD5 has no way to reverse a
hash back to `55`. That's the core tradeoff behind picking a reversible
algorithm (AES) vs. a one-way hash (MD5) per column, based on whether you
ever need the original value back or only need to match against it.

---

## Part 6: Combining Encryption with Sharding + Read-Write Splitting

Since `!ENCRYPT`, `!SHARDING`, and `!READWRITE_SPLITTING` are all just entries
in the same `rules:` list, they can be layered together in one file:

```yaml
rules:
  - !READWRITE_SPLITTING
    # ... (as in the previous guide)
  - !SHARDING
    # ... (as in the previous guide, referencing the readwrite-splitting logical names)
  - !ENCRYPT
    # ... (as shown above)
```

**Order of operations when all three are active on a write:**
1. **Encrypt** transforms the sensitive column values into ciphertext first.
2. **Sharding** then looks at the (now-encrypted, if the shard key itself is
   an encrypted column) or plain shard key value to decide the target
   database/table.
3. **Read-write splitting** finally decides the physical server (master, in
   this case, since it's a write) that actually receives the rewritten,
   sharded, encrypted `INSERT`.

**One important design note:** if you plan to shard by a column, be careful
about also encrypting that same column with a non-deterministic algorithm —
sharding needs a **consistent, comparable** value to compute something like
`value % 2` reliably. This is exactly why `assistedQuery` columns exist: if
you must both shard *and* encrypt on the same logical column, you'd typically
shard using a deterministic representation of the value (like the assisted
query hash) rather than the raw AES ciphertext, which isn't guaranteed to be
the same across encryptions.

---

## Summary

- **What it is:** transparent column-level encryption/decryption, applied by
  rewriting SQL before it reaches the database and decrypting results before
  they reach your app — no application code changes needed.
- **Core mechanic:** a logical column name maps to one or more real physical
  columns (`cipher`, optionally `assistedQuery` for exact lookups, optionally
  `likeQuery` for fuzzy search).
- **Reversible vs one-way:** pick AES (or similar) when you need the original
  value back; pick MD5 (or similar) when you only ever need to match against
  it, never read it back.
- **Combines cleanly** with sharding and read-write splitting since all three
  are just entries in the same `rules:` list — with the one caveat that
  sharding on an encrypted column needs a deterministic value to compute
  shard routing correctly.
