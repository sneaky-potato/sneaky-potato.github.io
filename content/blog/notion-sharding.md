+++
author = "sneaky-potato"
title = "Sharding databases: Lessons from Notion"
date = "2026-04-09"
description = "sharding postgres comes with some resposbilities"
tags = [
    "tech",
]
enableEmoji = true
+++

Decisions taken by Notion[^1] team to shard their massive postgres database:

## 1. Shard by ownership

Everything in Notion revolves around a `block`. They shard all data transitively 
related to a block.

Instead of
- `block` table - shard 1
- `comment` table - shard 2

They do: everything owned by block X - shard N.
The principle is data that must be updated together must live together.

**Why?**
Imagine we shard like this: one shard contains `block` and another contains
`comment`. Now if a user deletes a block, we need to update the comments.

```sql
DELETE block
also delete/update comments
```

**But** databases only guarantee transactions inside one shard, not across shards.

So say the user deletes a block, block delete succeeds in the shard, but
comment delete fails in other shard.
So to mitigate this, they decided to shard all data which can be reached from the
block table

```shell
Shard 1:
  Block A
  Discussion A
  Comments A

Shard 2:
  Block B
  Discussion B
  Comments B
```

This design enables transactions to be used, like so:
```sql
BEGIN TRANSACTION
DELETE block
DELETE comments
COMMIT
```

## 2. Capacity planning

They settled on an architecture consisting of **480** logical shards evenly distributed 
across **32** physical databases. The hierarchy looked like this:

- Physical database (32 total)
    - Logical shard, represented as a Postgres schema (15 per database, 480 total)
        - block table (1 per logical shard, 480 total)
        - collection table (1 per logical shard, 480 total)
        - space table (1 per logical shard, 480 total)

**Why 480?**
480 is divisible by a lot of numbers, which provides flexibility to add or remove
physical hosts while preserving uniform shard distribution. For example, in the
future we could scale from **32** to **40** to **48** hosts, making incremental
jumps each time.

By contrast, suppose we had 512 logical shards. The factors of 512 are all
powers of 2, meaning we’d jump from **32** to **64** hosts if we wanted to keep the
shards even. Any power of 2 would require us to double the number of physical
hosts to upscale. Pick values with a lot of factors!

## 3. Migration

- Double-write: Incoming writes get applied to both the old and new databases.
- Backfill: Once double-writing has begun, migrate the old data to the new database.
- Verification: Ensure the integrity of data in the new database.
- Switch-over: Actually switch to the new database. This can be done incrementally, e.g. double-reads, then migrate all reads.


**Audit log and catch-up script**: Create an audit log table to keep track of
all writes to the tables under migration. A catch-up process iterates through
the audit log and applies each update to the new databases, making any
modifications as needed.

They also prepared and tested a reverse audit log and script in case we needed to
switch back from shards to the monolith. This script would capture any incoming
writes to the sharded database, and allow them to replay those edits on the
monolith.

## 4. Verification

Before migrating read queries, they added a flag to fetch data from both old
and new databases (known as [dark reading](https://slack.engineering/re-architecting-slacks-workspace-preferences-how-to-move-to-an-eav-model-to-support-scalability/)).
The records are compared and the sharded copy is discarded, logging discrepancies
in the process. Introducing dark reads increases API latency, but provides
confidence that the switch-over would be seamless.

As a precaution, the **migration and verification logic were implemented by
different people**. Otherwise, there was a greater chance of someone making the
same error in both stages, weakening the premise of verification.

The link to the original blog can be found at the end of this article.

## 5. Re-sharding without downtime

Shortly after this the Notion team again needed to shard the data into more
databases[^2]. But this time they smartly rolled out the update without
requiring downtime.

### a. Set new shards

After capacity planning they decided to further triple the databases back in 2023.
So the number of instances in their fleet would go from 32 to 96 machines.

They used Terraform to automate the provisioning process and configured the new
databases with the smaller instance types and disks, since they now would each
need to handle much less traffic.

### b. Synchronization

I recently came across a powerful capability of PostgreSQL which is
[logical replication](https://www.postgresql.org/docs/current/logical-replication.html).
This means Postgres natively supports creating publishers, subscribers and creating
sync processes between them.

They used built-in Postgres logical replication to copy over the historical 
data and continuously apply new changes to the new databases. They wrote 
tooling to set up 3 Postgres publications on each existing database, 
covering 5 logical schemas apiece. 

All the tables were added from the respective logical schemas to the
publications. On the new databases, subscriptions were created to consume
one of the 3 publications, which copied over the relevant set of data.

Video explaining logical replication with a tutorial [here](https://www.youtube.com/watch?v=OvSzLjkMmQo)

---

[^1]: Part 1 of Notion's blog on sharding: [Herding elephants: Lessons learned from sharding Postgres at Notion](https://www.notion.com/blog/sharding-postgres-at-notion)
[^2]: Part 2 of Notion's blog on sharding: [The Great Re-shard: adding Postgres capacity (again) with zero downtime](https://www.notion.com/blog/the-great-re-shard)
