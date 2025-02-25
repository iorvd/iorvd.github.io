---
title: "Updating 300 millions of rows in PostgreSQL"
date: "2024-09-11"
aliases: ["/how-we-updated-300-millions-of-rows-in-postgresql"]
---

[*Originally posted at [DOU.ua](https://dou.ua/forums/topic/51676/)*]

This article was written as a retrospective of our experience migrating large amounts of data in PostgreSQL.
Data migration is a fairly non-trivial task, and as our team discovered, in a resource-limited
environment `UPDATE` of data is significantly more complex than `INSERT` or `DELETE`. We will
review the preconditions and kind of data we migrated, the requirements set for the migration,
explore the approaches we chose, and draw conclusions. This article will primarily
be useful to those planning to perform large data migrations and those who want to deepen
their knowledge of PostgreSQL.

# Preconditions

During a major refactoring of one of our services, we noticed some suboptimal data access patterns in
in our database and decided to optimise them. Worth noting that this was not the only reason why we decided to perform the migration,
but since this is all that can be revealed publicly, let's assume that all the reasons added up in
a reasonable decision to migrate the data.

Let's look at the example that mimics the behavior we had. There are
three entities — `user`, `transaction`, and `transaction_details`. The transaction details entity is
linked to the transaction, and the transaction is linked to the user. So in order to get user data by
transaction_details.id you first need to get the transaction detais, then get transaction,
and only then get the data from the user. Our migration updated many fields, but for simplicity, we will consider in the query examples only updating the user_id field in the transaction_details table by transferring its value from the transactions table.

![ERD](https://s.dou.ua/storage-files/unnamed_B5Sn67S.png)

A brief description of what we had and what our migration requirements were:
- The base in which migration was carried out was under constant load and served live traffic ~20rps with ocasional spikes up to 150rps.
![Traffic HTTP](https://s.dou.ua/storage-files/unnamed_PndVs3L.png)
<p style="text-align: center;">http traffic database served</p>

- We needed to reduce the load on the database as much as possible to avoid downtime of critical components of the app.
At the same time we did not have strict restrictions on the migration execution time (provided that this
did not require the use of engineers' working time). A faster solution was better for us than an ideal
solution which would involve more human resources.
- We did not have the opportunity to upgrade and then downgrade the database, but we added the maximum number of provisioned IOPS for our disk.
- The fields that we planned to update (`user_id`) were immutable in the target-table (`transaction_details`). And once the row was created, they remained unchanged, so we did not need to worry about data races.
- Zero data loss - we could not, for example, [stop the database](https://blog.codacy.com/how-to-update-large-tables-in-postgresql),
migrate the data, and then start database again.

# Preparation

First, we started filling the newly created fields with data. This pinned the total number of rows
across all the tables we needed to update to around 320 million, with the largest table containing
140 million records. Some of the configurational data we needed was in another database, but since
the data we needed was immutable, and we were not interested in new records (because they were
already automatically saturated with the data), in order to speed up the migration we completely
copied the required table from one database to another.


# Research on approaches to migration
## AWS DMS

We use AWS as a cloud provider for our services. One of AWS services is a service for data migration
between databases — DMS (Data Migration Service). AWS DMS allows you to migrate data between
homogeneous databases — Full Load and CDC, which consists of copying the main chunk of data in the
database and then applying all changes that have occurred in the old database via CDC. During this
migration, you can configure triggers that will modify the data — and as a result, you will get a ready
copy of the original database, but with all the necessary changes.

Advantages:

- migration script execution speed.
- if you plan to upgrade your database, you can perform both the upgrade and migration at the same time, saving time.

Disadvantages:

- all components must be thoroughly tested, as there is a high probability of data loss at any
migration stage or, conversely, the presence of data after migration that should have been
deleted in the original table.
- if we want to update the tables in several iterations, then each DMS migration will involve
a significant amount of infrastructure team engineering resources, which were limited in our situation.
- preparing to migrate all the tables at once is more difficult, since in the event of an error during
the migration, it will have to be started from the beginning.


## Own wheen-reinvented DMS
If the approach of migrating the main chunk and adding data via CDC works, but there is no need to
upgrade the database, then why use DMS if you can perform the same operations on the existing database?
Let's consider an example of what such a migration could look like:

```sql
-- remember the latest_updated_at;
SELECT updated_at FROM origin_table ORDER BY updated_at DESC LIMIT 1;
-- copy the table either the following way or via pg_dump/pg_restore,
-- and enrich the exported data locally;
CREATE TABLE origin_table_copy FROM (SELECT * FROM origin_table);
-- create index on id;
-- create trigger on create/update/delete, which will write from
-- origin_table to origin_table_copy;
-- incrementally restore the modified data;
UPDATE origin_table_copy
SET x=y
WHERE updated_at BETWEEN latest_updated_at
    AND latest_updated_at + (10 * interval '1 minute');
-- restore all the constraints, recreate indexes, restart sequences etc;
-- switch traffic between the tables;
```

Advantages:

- speed of migration script execution.
- easier to replicate the migration environment, and as a result, easier to test individual components than in DMS
- less involvement of infrastructure engineers
- no problems with individual table migration.

Disadvantages:

- at the same time, there are significantly more components to test than in DMS.
Also, the logic of the migration itself is more complex, and accordingly, there
is more room for a mistake.

## Batch updates

The idea behind updating data in batches is that it allows for a quick migration
given the availability of database resources and a well-optimized query. In particular,
this method is the easiest in terms of mental overhead for engineers, as it is intuitive
in understanding. And therefore it helps to avoid mistakes.

Initially, we tried a fairly trivial and straightforward approach — to aggregate all
the data in one sql transaction and immediately update the target table.

```sql
CREATE INDEX tx_details_user_id_null_idx
    ON transaction_details (id)
    WHERE user_id IS NULL

UPDATE transaction_details td
SET user_id = cte.user_id
FROM (SELECT t.user_id, td.id
      FROM transactions t
  JOIN transaction_details td ON td.transaction_id=t.id
      WHERE td.user_id IS NULL
      LIMIT _limit
) AS cte
WHERE td.id = cte.id;
```

This worked very slowly, the speed was about 3k rows/min. Simple math suggests that at
this rate, migrating one table would take **~50 days** (assuming we would turn
the migration off at night). The first problem with the query above is that we waste the
time JOINing the tables. To avoid this, we created an auxiliary table to which we added
pre-aggregated data:

```sql
-- join all the data we need and put it in a table
CREATE TABLE tmp AS
SELECT ROW_NUMBER() OVER() AS row_id,
       td.id AS td_id,
       t.id AS t_id
FROM transaction_details td
JOIN transactions t ON td.transaction_id = t.id
WHERE td.user_id IS NULL;

-- adding an index on top of it
CREATE INDEX tmp_idx ON tmp(row_id);

-- table with all the data we need
UPDATE transaction_details td
SET user_id = COALESCE(td.user_id, cte.user_id)
FROM (SELECT user_id,
             td_id
      FROM tmp
      WHERE row_id > 200000 AND row_id <= 210000) AS cte
WHERE td.td_id = cte.td_id;
```

The second problem we encountered was less obvious, but no less significant.
The thing is that the `tx_details_user_id_null_idx` index does not speed up,
but in fact slows down UPDATE, since when updating table rows, this index
also needs to be updated. Thus, the advantages of the fast table reads are
shadowed by the slow index updates. Therefore, we dropped it. After that the data was updating
at ~5k rows/min rate, but we still were not satisfied with the speed.

We tried a few more rather bold ideas: vacuuming dead tuples after each iteration (it had almost no effect),
playing with the postgres settings `temp_buffers`, `work_mem`, `effective_cache_size` (also minimal impact).
In the end, after several iterations of changes, we came to the final version, the speed of which
satisfied us **~30k rows/min**. Let's take a look at how we did it.

First, we created a table in which we aggregated all the data we need:

```sql
-- join all the data we need and put it in a table
CREATE TABLE tmp AS
SELECT td.id         AS td_id,
       t.user_id         AS user_id,
       td.updated_at AS updated_at
FROM transaction_details td
JOIN transactions t ON t.id = td.transaction_id
WHERE td.user_id IS NULL;

-- create an index to read fast first N records and the delete them
CREATE INDEX tmp_idx ON tmp (updated_at, td_id) INCLUDE (user_id)
```

We found that an important aspect is sorting the index by updated_at. When updating data,
our ideal scenario is to read all the data from the tables that will be updated
in the same order in which they reside on disk. This allows us to reduce the read overhead on it.
In PostgreSQL, the `UPDATE` operation is `INSERT+DELETE`, so when updating a row, its new versions
can be written to a different memory region than the current one. As a result, the probability that
rows with close `updated_at` values ​​are close to each other is higher than any two arbitrary
rows sorted by UUID.

It is worth noting that this should be `updated_at`, not `created_at`. If the table [fillfactor](https://www.postgresql.org/docs/16/sql-createtable.html#RELOPTION-FILLFACTOR) is
100% (the default value), then data updates with a high probability will be put on another table page,
and the adjacent data risks be put far from each other after creation. We also tried sorting by
ctid, but, strangely enough, it turned out to be much slower compared with updated_at.

When data is updated, indexes that reference the old version of that data are also updated.
Therefore, it is a good idea to drop as many [indexes](https://www.postgresql.org/docs/16/indexes-intro.html) of the updated table as possible
and recreate them after the migration. We have observed ~10%-30% improvement in the speed of `UPDATE`
operations after removing several large unused indexes.

For example, the query execution speed on a database clone with and without any indexes:

 <table>
  <tr>
    <th></th>
    <th>With indexes</th>
    <th>W/o indexes</th>
  </tr>
  <tr>
    <td>Batch size 1000</td>
    <td>36 000</td>
    <td>111 000</td>
  </tr>
</table>

To reduce the overhead of updates, MVCC might use [HOT](https://www.postgresql.org/docs/16/storage-hot.html) (Heap-Only-Tuples) optimization,
but in order for it to happen the update must meet certain conditions:

- the update does not modify columns used in table indexes.
- there is enough space on the table page that contains the old version of the row to fit the new version.

And the second bullet above is the responsibility of the fillfactor - so if update-heavy workload dominates for a table
it is worth thinking about creating tables with a fillfactor <70-90%. This will improve
the speed of data updates during any migration as well.

At the end, our migration looked like this:

```sql
DO $$
    DECLARE
        _id int := 0;
        _rowsLimit INT := 3000;
        _updatedTotal INT := 0;
        _updatedInBatch INT;
        start_time timestamp := CLOCK_TIMESTAMP();
        update_time INT;
        execution_time INT;
    BEGIN
        -- this setting is needed to turn off the trigger on the target table
        SET session_replication_role='replica';

        LOOP
            RAISE NOTICE 'Started Iteration: %', _id;
            -- these rows from cte could be written into a variable and later
            -- reused for delete-operation, but we didn't think of it at the time,
            -- because delete was not our bottleneck even remotely.
            WITH cte AS (SELECT td_id, user_id
                         FROM tmp
                         ORDER BY updated_at
                         LIMIT _rowsLimit
            )
            UPDATE transaction_details td
            SET user_id = COALESCE(td.user_id, cte.user_id)
            FROM cte
            WHERE td.id = cte.td_id;

            COMMIT;

            GET DIAGNOSTICS _updatedInBatch = ROW_COUNT;
            _updatedTotal := _updatedTotal + _updatedInBatch;
            update_time := EXTRACT(EPOCH FROM clock_timestamp() - start_time);
            RAISE NOTICE 'UPDATE executed time: % sec.', update_time;

            WITH cte AS (SELECT td_id, updated_at
                         FROM tmp
                         ORDER BY updated_at
                         LIMIT _rowsLimit
            )
            DELETE FROM tmp t
                USING cte
            -- comparing `updated_at` is needed to force the query planner
            -- to use the index.
            WHERE t.updated_at = cte.updated_at
              AND t.td_id = cte.td_id;

            execution_time := EXTRACT(EPOCH FROM clock_timestamp() - start_time);
            RAISE NOTICE 'Finished Iteration: %, updated total: %, ALL time: % sec.', _id, _updatedTotal, execution_time;

            COMMIT;

            IF _updatedInBatch = _rowsLimit THEN
                PERFORM pg_sleep(0.5);
            ELSE
                RAISE NOTICE 'All IDs were updated. Exit.';
                EXIT;
            END IF;

            _id := _id+1;
        END LOOP;
    END $$;
```

# Testing

The query planner takes into account the state of the table and the database,
so any query analysis should be performed on real data in the production database.
Since there is always a risk of breaking something in production, we spinned up
a clone of our production database and performed all hypothesis testing on it.

Additionally, we tested the migration segments via functional tests to make sure
that they do exactly what we want them to do. Before executing the migration in production,
we ran it in our test environments and verified that the data was migrated correctly.

# Monitoring and performance tuning

When writing and executing migration, it is important to remember about domain-specific data access patterns.

- Are traffic spikes typical for services that work with the database? If so, you should consider reducing the speed of data migration.
- Are there data synchronizations that create additional load on the database at night? Perhaps it is worth disabling migration at night.
- How and when is data unloading for DWH or cache and can migration affect it? We had some CDC logic built on triggers - we had to disable it so as not to burden the services that use it (about 20 million rows were updated every day).
- How long will it take to update one batch? Since thousands of values ​are being updated ​in the batch, each row will be locked with `FOR UPDATE` lock. Can this negatively affect other concurrent queries?
- Are there any entities that use the database/table, the downtime of which directly affects the company's revenue? For us, such entities were endpoints that are responsible for conducting payments. During migration, it is important to identify such sensitive places and monitor whether requests are not falling due to timeout.
- Who and how monitors the migration process and who sits near the kill switch script in case of an incident threat?
- After migration, a significant number of dead tuples will be created and it is a good practice to clean them up using VACUUM ANALYZE.

# Conclusions

Over time, you may need to re-saturate old data with new data or reshape the structure of tables, but as your database grows,
this task becomes increasingly non-trivial. Increased query latency, disk degradation,
data loss/data races — all of these problems appear to engineers in a new light
when it comes to large amounts of data, and performing routine manipulations requires
increasingly creative approaches and more time for preparation.

It is important to conduct more thorough and meticulous research on how you plan
to perform the migration — from what script you will run to what metrics you will be
able to look at if services dependent on the database begin to degrade.
