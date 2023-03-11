---
date: "2023-03-10 12:00"
title: "Setting up MySQL Aurora replication from an external primary with GTIDs "
---

Setting up Amazon Aurora as a replica of an external MySQL primary is a common way of synchronizing and/or migrating self-managed MySQL databases to RDS/Aurora.

Nowadays, using [GTIDs for MySQL](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids.html) is the preferred way of doing replication, for example, because it offers features such as auto-position and thus makes it easy to change replication topologies.

According to the [AWS docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/mysql-replication-gtid.html) and also [AWS blogs](https://aws.amazon.com/blogs/database/migrating-to-amazon-aurora-mysql-with-fallback-option-using-gtid-based-replication/), it’s possible to use GTID for replication from an external primary into Aurora.

But there’s one important issue which is not covered by the docs or blogs, and this is [setting the gtid_executed / gtid_purged variables](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids-failover.html#replication-gtids-failover-gtid-purged), which new MySQL replicas usually need GTIDs, so they know their initial binlog position. This value is not set in Aurora/RDS, at least when restoring from a Xtrabackup on S3. Also, RDS/Aurora does not allow setting this value:

```
mysql> SET GLOBAL gtid_purged='<gtid_string_found_in_xtrabackup_binlog_info>';

Access denied; you need (at least one of) the SUPER privilege(s) for this operation
```

When I started the replication without the GTID set, it immediately stopped working with obscure errors, since it was apparently starting to replicate events from the primary at a random position, such as:

```
mysql> show slave status\G
*************************** 1. row ***************************
...
Last_Errno: 1062
Last_Error: Could not execute Write_rows event on table sampletable; Duplicate entry '102688557' for key 'PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000001, end_log_pos 1099
```
But, AWS Support to the rescue, knew a workaround by setting the gtid_executed value directly in the `mysql` database:

```
mysql> insert into mysql.gtid_executed(source_uuid,interval_start,interval_end) values('5f70944c-9bbe-11e9-a9d2-0a75ff943724',1,19);
```

Note: This also works with a set of multiple GTIDs. Just insert more rows)

Now, reboot the DB instance. Check the value has been set into the `gtid_purged` variable:

```
mysql> show variables like 'gtid_purged';
```

And, if it looks correct, start the replication:

```
mysql> CALL mysql.rds_reset_external_master;
mysql> CALL mysql.rds_set_external_master_with_auto_position ('..., '3306', username, passwordX', 0);
CALL mysql.rds_start_replication();
```

Note: These commands are for Aurora 2/MySQL 5.7. Aurora 3 / MySQL 8 have naming changes to a more inclusive language, the commands [look slightly different](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Replication.MySQL.html).

After setting the correct initial value for the `gtid_executed` value, the replication should be running smoothly and catching up. 
