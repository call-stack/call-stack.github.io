---
layout: post
title:  "Postgres Partitioning Without Downtime"
date:   2024-04-13 10:54:35 +0530
categories: jekyll update
---
## Introduction
Managing large tables in a PostgreSQL database can present significant challenges, especially when performance begins to degrade due to the sheer volume of data. In such cases, partitioning the table can offer a solution by dividing it into smaller, more manageable chunks. Partitioning not only improves query performance but also facilitates data management tasks such as archiving and purging.

In our payment system, we have a critical `transactions` table that stores every transaction processed by our system. With an average of `100 requests per second`, the table grows by approximately `30GB per day`, leading to performance and storage challenges.
In this blog post, we will delve into the process of partitioning this substantial **5TB** table in PostgreSQL without experiencing no to minimal downtime. Our approach focuses on utilizing `range partitioning` with the `created_at` column as the partitioning key. By partitioning based on the creation timestamp, we can efficiently segregate older data into separate partitions, enabling us to move it to cold storage while retaining fast access to recent data.

Partitioning a large table without downtime requires careful planning and execution to ensure uninterrupted access to the database. We will explore the steps involved in preparing for and executing the partitioning process seamlessly. Additionally, we will discuss best practices and considerations for long-term maintenance to ensure optimal performance and manageability of the partitioned table.

## Understanding Postgres Partitioning
Before diving into the specifics of partitioning the transactions table, let's take a moment to understand what database partitioning is and how it works in PostgreSQL.

Database partitioning is a technique that involves splitting a large table into smaller, more manageable pieces called partitions. Each partition contains a subset of the data based on a specified criteria, such as a date range or a category. Partitioning can help improve query performance, simplify data management, and enable more efficient data archival and deletion.

PostgreSQL supports several types of partitioning:

1. **Range Partitioning**: This type of partitioning is based on a range of values in a specific column. For example, partitioning a table by date ranges, such as creating separate partitions for each month or year.

2. **List Partitioning**: List partitioning allows you to define partitions based on a list of discrete values in a column. For instance, partitioning a table based on different categories or status values.

3. **Hash Partitioning**: Hash partitioning distributes data across partitions based on the hash value of a specified column. This type of partitioning is useful for evenly distributing data across partitions.

In our case, we decided to use range partitioning based on the `created_at` column, which represents the timestamp of each transaction. By partitioning the `transactions` table by date ranges, we can easily manage the data lifecycle and archive older partitions that are no longer needed for active use.

When a table is partitioned, PostgreSQL creates a parent table that acts as an empty shell. The actual data is stored in child tables, which are the individual partitions. Queries on the parent table are automatically routed to the appropriate child tables based on the partition key.

One of the key benefits of partitioning is improved query performance. By dividing the data into smaller partitions, queries that filter on the partition key can target only the relevant partitions, reducing the amount of data scanned and speeding up the query execution.

Another advantage of partitioning is simplified data management. With partitioning, you can perform maintenance tasks, such as vacuum and analyze, on individual partitions rather than the entire table. This allows for more efficient resource utilization and reduces the impact on the overall system.

However, partitioning also comes with some considerations and trade-offs. It requires careful planning and design to ensure that the partitioning scheme aligns with your query patterns and data distribution. Additionally, partitioning introduces some overhead in terms of maintenance and query planning.

## Preparing for Partitioning
Before we dive into the actual partitioning process, let's take a look at the DDL of our `transaction` table:
```sql
CREATE TABLE public.transaction (
  id int8 NOT NULL DEFAULT id_generator(),
  ref_id text NULL,
  txn_ref_id text NULL,
  msg_id text NULL,
  biller_id text NULL,
  api text NULL,
  request_payload jsonb NULL,
  response_payload jsonb NULL,
  status text NULL,
  created_at timestamp NOT NULL DEFAULT current_timestamp_utc(),
  modified_at timestamp NOT NULL DEFAULT current_timestamp_utc(),
  is_deleted bool NULL DEFAULT false,
  CONSTRAINT transaction_pkey PRIMARY KEY (id)
);

CREATE INDEX transaction_ref_id ON public.transaction USING btree (ref_id);
CREATE INDEX txn_ref_api_idx ON public.transaction USING btree (txn_ref_id, api);

-- Table Triggers
CREATE TRIGGER set_timestamp_txn
BEFORE UPDATE ON public.transaction
FOR EACH ROW
EXECUTE FUNCTION trigger_set_timestamp_modify();
```

Our strategy for partitioning without downtime involves using the existing table as a child table of a new partitioned table. To achieve this, we need to ensure that the table meets certain requirements and constraints. Let's discuss the necessary steps:

1. **Primary Key and Partitioning Key**: In PostgreSQL, a partitioned table requires that the primary key includes the partitioning key. This is important because it guarantees data uniqueness across all partitions. Without the partitioning key in the primary key, there could be duplicates across different child tables, violating the integrity of the data. For example, let's say we have a partitioned table `orders` with a **primary key** on the `order_id` column and a **partitioning key** on the `created_at` column. If we insert two orders with the same `order_id` but different `created_at` values into different partitions, it would violate the uniqueness constraint of the primary key. To address this, we need to modify the primary key of our transaction table to include the partitioning key column `(created_at)`.
2. **Data Constraints Check**: When attaching an existing table as a child table of a **partitioned table**, if the new partition is a regular table, PostgreSQL performs a **full table scan** to check that existing rows in the table do not violate the partition constraint. This check can be time-consuming for large tables and may block other operations. To mitigate this issue, we need to find a way to handle the data constraints check before attaching the old table as a partition.

Now, let's discuss the steps we took to prepare for partitioning and handle above cases:

1. **Adding the Partitioning Key to the Primary Key**: Since our existing `transaction` table does not include the `created_at` column in its **primary key**, we need to add it. However, modifying the primary key directly can be a blocking operation. To work around this, we can create a new **unique constraint** that includes the existing primary key column `(id)` along with the partitioning key column `(created_at)`. This unique constraint can be added concurrently, without blocking other operations.
```sql
CREATE UNIQUE INDEX CONCURRENTLY transaction_old_unique_idx ON transaction (id, created_at);
```
By adding this unique constraint, we ensure that the combination of the `id and created_at` columns is unique across the table. Later, when we attach the old table as a partition, we can promote this unique constraint to the primary key.

2. **Handling Data Constraints Check**: To avoid the blocking data constraints check when attaching the old table as a partition, we can use a `CHECK` constraint with the `NOT VALID` option. Since using the `ALTER TABLE` command with a `CHECK` constraint directly would block the table, we specify the `NOT VALID` option. This prevents validation of existing data (the constraint will still be enforced against subsequent inserts or updates) and allows other updates to the table to proceed without being locked out.
```sql
ALTER TABLE transaction_old ADD CONSTRAINT skip_check_range CHECK (created_at >= '2021-02-23 00:00:00.000' AND created_at < '2023-04-01 00:00:00.000') NOT VALID;
```
After adding the `CHECK` constraint with `NOT VALID`, we need to validate it before attaching the old table as a partition. We can use the following command:
```sql
ALTER TABLE transaction_old VALIDATE CONSTRAINT skip_check_range;
```
This command validates the constraint created above and takes a `SHARE UPDATE EXCLUSIVE` lock, which won't block reads or writes on the table.

By performing these preparatory steps, we ensure that our `transaction` table is ready to be partitioned without causing downtime or blocking other operations.

In the next section, we will walk through the actual process of partitioning the table and attaching the existing table as a child partition.

## Partitioning the Table
On the day of partitioning our transaction table, we execute the following operations in a single transaction to ensure data consistency and minimize downtime:

1. Create the new partitioned table: 
   ```sql
   CREATE TABLE public.transaction_partitioned (
  id int8 NOT NULL DEFAULT id_generator(),
  ref_id text NULL,
  txn_ref_id text NULL,
  msg_id text NULL,
  biller_id text NULL,
  api text NULL,
  request_payload jsonb NULL,
  response_payload jsonb NULL,
  status text NULL,
  created_at timestamp NOT NULL DEFAULT current_timestamp_utc(),
  modified_at timestamp NOT NULL DEFAULT current_timestamp_utc(),
  is_deleted bool NULL DEFAULT false) PARTITION BY RANGE(created_at);
   ```
   This statement creates a new table named `transaction_partitioned` with the same structure as the existing transaction table. The PARTITION BY RANGE(created_at) clause specifies that the table will be partitioned based on the `created_at` column.

2. Add the primary key constraint to the partitioned table:
```sql
ALTER TABLE transaction_partitioned ADD CONSTRAINT transaction_partitioned_pkey PRIMARY KEY (id, created_at);
```
This statement adds a primary key constraint to the `transaction_partitioned` table, using the combination of the `id and created_at` columns. This ensures data uniqueness across all partitions.

3. Create necessary indexes on the partitioned table: 
   ```sql
    CREATE INDEX transaction_partitioned_ref_id ON transaction_partitioned (ref_id);
    CREATE INDEX transaction_partitioned_txn_ref_id_api ON transaction_partitioned (txn_ref_id, api);
   ```
   These statements create **indexes** on the `ref_id` and the combination of `txn_ref_id and api` columns in the transaction_partitioned table. These indexes will be used to optimize query performance on the partitioned table.

4. Rename the existing table: 
    ```sql
    ALTER TABLE transaction RENAME TO transaction_old;
    ```
    This statement renames the existing `transaction` table to `transaction_old`. This step is necessary to attach the old table as a partition to the new table later.
5. Rename the partitioned table: 
    ```sql
    ALTER TABLE transaction_partitioned RENAME TO transaction;
    ```
    This statement renames the `transaction_partitioned` table to `transaction`. This step ensures that the partitioned table takes the place of the original table, minimizing the impact on existing queries and applications.

6. Drop the old primary key constraint and add the new one:
    ```sql
    ALTER TABLE transaction_old DROP CONSTRAINT transaction_pkey CASCADE;
    ALTER TABLE transaction_old ADD CONSTRAINT transaction_old_pkey PRIMARY KEY USING INDEX transaction_old_unique_idx;
    ```
    These statements drop the existing **primary key** constraint on the transaction_old table and add a new primary key constraint using the previously created unique index `transaction_old_unique_idx`, which includes the `id and created_at` columns.
7. Attach the old table as a partition: 
   ```sql
   ALTER TABLE transaction ATTACH PARTITION transaction_old FOR VALUES FROM ('2021-02-23 00:00:00') TO ('2023-04-01 00:00:00');
   ```
   This statement attaches the `transaction_old` table as a partition of the `transaction` table for the specified range of `created_at` values.
   
8. Update triggers:
    ```sql
    DROP TRIGGER set_timestamp_txn ON transaction_old;
    CREATE TRIGGER set_timestamp_txn BEFORE UPDATE ON transaction FOR EACH ROW EXECUTE FUNCTION trigger_set_timestamp_modify();
    ```
    These statements drop the existing `set_timestamp_txn` trigger on the `transaction_old` table and recreate it on the `transaction` table. This ensures that the trigger is associated with the partitioned table.

We observed that attaching the existing table as a child table of the new partitioned table, along with the **primary key** on `id and created_at`, took only **2 seconds**. This demonstrates the efficiency of the partitioning process and minimizes the impact on the system's availability.

After executing these steps, our transaction table is now a partitioned table based on the `created_at` column. This allows us to efficiently manage and query the data based on time ranges.

## Testing and Validation
After successfully partitioning the transaction table, it's crucial to thoroughly test and validate the partitioned table to ensure data integrity, query performance, and the correct behavior of partition pruning.

To validate the effectiveness of partitioning, let's run a sample query that filters on the `ref_id` and `created_at` columns:

```sql
EXPLAIN ANALYZE
SELECT *
FROM transaction
WHERE ref_id = 'qUzbUvh1av41041234' AND created_at >= '2024-04-10 00:00:00' AND created_at < '2024-04-12 00:00:00';
```
Here's the resulting execution plan:

```
Index Scan using transaction_p2024_04_ref_id_idx on transaction_p2024_04 transaction (cost=0.57..8.59 rows=1 width=1544) (actual time=9.847..9.847 rows=0 loops=1)
  Index Cond: (ref_id = 'qUzbUvh1av41041234'::text)
  Filter: ((created_at >= '2024-04-10 00:00:00'::timestamp without time zone) AND (created_at < '2024-04-12 00:00:00'::timestamp without time zone))
  Rows Removed by Filter: 1
Planning Time: 1.045 ms
Execution Time: 9.872 ms
```
Let's analyze the execution plan:
1. PostgreSQL is performing an **"Index Scan"** operation using the index `transaction_p2024_04_ref_id_idx` on the partition `transaction_p2024_04`. This indicates that PostgreSQL has identified the relevant partition based on the query conditions.
2. The **"Index Cond"** line shows that PostgreSQL is using the index to search for a specific ref_id value, which is **qUzbUvh1av41041234** in this case.
3. The **"Filter"** line indicates that PostgreSQL is applying an additional filter condition on the `created_at` column, ensuring that it falls within the specified range `('2024-04-10 00:00:00' to '2024-04-12 00:00:00')`.
4. The **"Rows Removed by Filter"** line shows that one row was initially matched by the index condition but was subsequently removed by the filter condition on created_at. This suggests that there was a row with the matching ref_id, but it didn't fall within the specified date range.
5. The **"Planning Time"** and **"Execution Time"** lines provide information about the time taken for query planning and execution, respectively.

This execution plan confirms that PostgreSQL is efficiently using partition pruning to scan only the relevant partition (`transaction_p2024_04`) based on the query conditions. It is using the index `transaction_p2024_04_ref_id_idx` to search for the specific `ref_id` value and then applying the additional filter condition on `created_at` to narrow down the results.

It's important to note that we have chosen to partition the transaction table on a monthly basis, meaning that each partition represents one month of data. Ideally, we could have considered partitioning at a smaller interval, such as daily or weekly, depending on the specific use case and query patterns. However, our current use case and requirements led us to choose monthly partitioning as the most suitable approach.

To manage the creation of new partitions and the maintenance of older partitions, we are utilizing the `pg_partman` extension. pg_partman simplifies the process of creating and managing partitions based on predefined rules. It automatically creates new partitions as needed and can handle the retention and archiving of older partitions. I will cover the details of using pg_partman for partition management in a separate blog post.

By thoroughly testing and validating the partitioned table with different query patterns and analyzing the execution plans, we can ensure that partitioning is providing the expected performance benefits and that the data remains consistent and accessible.

## Lessons Learned and Best Practices
During the process of partitioning the transaction table, we encountered a few challenges and learned valuable lessons that are worth sharing. These lessons can serve as best practices for anyone undertaking a similar partitioning effort.

1. **Validating Unique Indexes**: One of the critical steps in partitioning is creating a **unique index** that includes the **partitioning key**. In our case, we created a unique index concurrently on the transaction table. However, it's important to note that when creating a unique index concurrently, there is a possibility that PostgreSQL may not be able to enforce the uniqueness constraint, rendering the **index invalid**. Before proceeding with partitioning, it's crucial to validate the created unique index to ensure its validity. You can run the following command to check for invalid indexes:
   ```sql
   SELECT * FROM pg_class, pg_index WHERE pg_index.indisvalid = false AND pg_index.indexrelid = pg_class.oid;
   ```
   If the query returns any results, it indicates that the index is invalid, and you need to address the issue before promoting the index to a primary key.
2. **Matching Check Constraints with Range Constraints**: When attaching a partition to the partitioned table, it's essential to ensure that the check constraint on the partition matches the range constraint specified in the `ATTACH PARTITION` command. If there is a mismatch, PostgreSQL will perform a lengthy table scan, even if you have previously validated the constraint. For example, consider the following command:
```sql
ALTER TABLE transaction ATTACH PARTITION transaction_old FOR VALUES FROM ('2021-02-23 00:00:00.000') TO ('2023-03-31 00:00:00.000');
```
If the upper limit of the range constraint (`'2023-03-31 00:00:00.000'`) does not match the upper limit of the check constraint on the transaction_old partition, PostgreSQL will resort to a full table scan, impacting performance. To avoid this issue, double-check that the check constraint on the partition aligns perfectly with the range constraint specified in the `ATTACH PARTITION` command.
3. **Testing on a Separate Environment**: Before executing the partitioning process in a production environment, it's highly recommended to set up a separate test environment that closely resembles the production setup. Use a snapshot of the production data created after **"Preparing for Partitioning"** section, to create a realistic test dataset. Run your migration script, as outlined in the **"Partitioning the Table"** section, on the test environment. This will allow you to validate the entire partitioning process, including the creation of unique indexes, check constraints, and the attachment of partitions. Testing in a separate environment helps identify any potential issues or errors that may arise during the partitioning process. It provides an opportunity to fine-tune the migration script, validate the results, and ensure a smooth execution when applying the same steps to the production environment.

## Conclusion
Partitioning large tables in PostgreSQL is a powerful technique for improving query performance, manageability, and data organization. By proactively partitioning tables before they become excessively large, we can avoid potential performance issues and ensure system scalability.

In this blog post, I shared my experience partitioning a large transaction table using a monthly partitioning strategy based on the `created_at` column. We discussed the steps involved, including preparing the table, executing the partitioning process, and attaching the existing table as a partition.

Testing and validation played a crucial role in ensuring the success of partitioning. Continuous monitoring and adjustments are necessary to maintain optimal performance and manage the partitioned table effectively.

Partitioning enables the implementation of data retention policies and efficient archiving of older data. It allows us to focus on the most relevant data while optimizing storage costs and query performance.

The lessons learned and best practices shared in this blog post can guide others in their partitioning endeavors. By carefully planning and executing the partitioning process, organizations can unlock the full potential of their PostgreSQL databases.

As we continue to work with the partitioned transaction table, we look forward to the benefits of improved performance and simplified data management. I encourage readers to explore the possibilities of partitioning in their own PostgreSQL environments and experience its benefits on large-scale data management.

## References
1. PostgreSQL Documentation - Partitioning: [https://www.postgresql.org/docs/current/ddl-partitioning.html](https://www.postgresql.org/docs/current/ddl-partitioning.html)
2. PostgreSQL - Check Constraints: [https://www.postgresqltutorial.com/check-constraint](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-CHECK-CONSTRAINTS)
3. PostgreSQL - Not Valid Option: [https://www.postgresqltutorial.com/not-valid-option](https://www.postgresql.org/docs/current/sql-altertable.html#SQL-ALTERTABLE-DESC-ADD-TABLE-CONSTRAINT)
4. PostgreSQL - Table Level Locks: [https://www.postgresqltutorial.com/table-level-locks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-TABLES)