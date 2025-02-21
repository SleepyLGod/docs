---
title: Batch Create Table
summary: Learn how to use the Batch Create Table feature. When restoring data, BR can create tables in batches to speed up the restore process.
---

# Batch Create Table

When restoring data, Backup & Restore (BR) creates databases and tables in the target TiDB cluster and then restores the backup data to the tables. In versions earlier than TiDB v6.0.0, BR uses the [serial execution](#implementation-principles) implementation to create tables in the restore process. However, when BR restores data with a large number of tables (nearly 50000), this implementation takes much time on creating tables.

To speed up the table creation process and reduce the time for restoring data, the Batch Create Table feature is introduced in TiDB v6.0.0. This feature is enabled by default.

> **Note:**
>
> - To use the Batch Create Table feature, both TiDB and BR are expected to be of v6.0.0 or later. If either TiDB or BR is earlier than v6.0.0, BR uses the serial execution implementation.
> - Suppose that you use a cluster management tool (for example, TiUP), and your TiDB and BR are of v6.0.0 or later versions, or your TiDB and BR are upgraded from a version earlier than v6.0.0 to v6.0.0 or later. In this case, BR enables the Batch Create Table feature by default.

## Usage scenario

If you need to restore data with a massive amount of tables, for example, 50000 tables, you can use the Batch Create Table feature to speed up the restore process.

For the detailed effect, see [Test for the Batch Create Table Feature](#test-for-the-batch-create-table-feature).

## Use the Batch Create Table feature

BR enables the Batch Create Table feature by default, with the default configuration of `--ddl-batch-size=128` in v6.0.0 or later to speed up the restore process. Therefore, you do not need to configure this parameter. `--ddl-batch-size=128` means that BR creates tables in batches, each batch with 128 tables.

To disable this feature, you can set `--ddl-batch-size` to `0`. See the following example command:

{{< copyable "shell-regular" >}}

```shell
br restore full -s local:///br_data/ --pd 172.16.5.198:2379 --log-file restore.log --ddl-batch-size=0
```

After this feature is disabled, BR uses the [serial execution implementation](#implementation-principles) instead.

## Implementation principles

- Serial execution implementation before v6.0.0:

    When restoring data, BR creates databases and tables in the target TiDB cluster and then restores the backup data to the tables. To create tables, BR calls TiDB internal API first, and then processes table creation tasks, which works similarly to executing the `Create Table` statement by BR. The TiDB DDL owner creates tables sequentially. Once the DDL owner creates a table, the DDL schema version changes correspondingly and each version change is synchronized to other TiDB DDL workers (including BR). Therefore, when BR restores a large number of tables, the serial execution implementation is time-consuming.

- Batch create table implementation since v6.0.0:

    By default, BR creates tables in multiple batches, and each batch has 128 tables. Using this implementation, when BR creates one batch of tables, the TiDB schema version only changes once. This implementation significantly increases the speed of table creation.

## Test for the Batch Create Table feature

This section describes the test information about the Batch Create Table feature. The test environment is as follows:

- Cluster configurations:

    - 15 TiKV instances. Each TiKV instance is equipped with 16 CPU cores, 80 GB memory, and 16 threads to process RPC requests ([`import.num-threads`](/tikv-configuration-file.md#num-threads) = 16).
    - 3 TiDB instances. Each TiDB instance is equipped with 16 CPU cores, 32 GB memory.
    - 3 PD instances. Each PD instance is equipped with 16 CPU cores, 32 GB memory.

- The size of data to be restored: 16.16 TB

The test result is as follows:

```
‘[2022/03/12 22:37:49.060 +08:00] [INFO] [collector.go:67] ["Full restore success summary"] [total-ranges=751760] [ranges-succeed=751760] [ranges-failed=0] [split-region=1h33m18.078448449s] [restore-ranges=542693] [total-take=1h41m35.471476438s] [restore-data-size(after-compressed)=8.337TB] [Size=8336694965072] [BackupTS=431773933856882690] [total-kv=148015861383] [total-kv-size=16.16TB] [average-speed=2.661GB/s]’
```

From the test result, you can see that the average speed of restoring one TiKV instance is as high as 181.65 MB/s (which equals to `average-speed`/`tikv_count`）.