---
date: "2023-02-19 12:00"
title: "Lessons learned when restoring a MySQL Aurora RDS database from S3/Percona Xtrabackup"
---

Recently I was trying to restore a Aurora database from an Percona xtrabackup, the de-facto industry standard for backing up self-managed MySQL databases. Luckily, RDS and Aurora natively support restoring a cluster from Percona xtrabackups. This comes very handy for migrations of big databases  (For more information, check out [the docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Migrating.ExtMySQL.html#AuroraMySQL.Migrating.ExtMySQL.S3) and [this prescriptive guidance article from AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/migrate-on-premises-mysql-databases-to-aurora-mysql-using-percona-xtrabackup-amazon-efs-and-amazon-s3.html)).

<!--more-->

But soon I was stuck with this error message:

```
Failed to migrate from mysql 5.7.38 to aurora-mysql 5.7.mysql_aurora.2.11.0. Reason: Migration has failed due to issues. Common problems include issues with the underlying storage or corruption in the database. Disabled/Deleted source kms key for migration from encrypted source. Try taking another snapshot and migrating again.
```
I contacted AWS support and luckily got a very knowledgeable contact person (thanks, JP!). They found out that the error message is misleading, since I was not restoring from a 5.7.38 backup, but from 5.7.41.

In the end, the problem was an incompatible xtrabackup format with a combination of xbstream and xbcloud. The error message probably (misleadingly) says “5.7.38” since this is the [current default minor version for RDS/MySQL 5.7](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Concepts.VersionMgmt.html).

A plain xtrabackup, split in chunks, worked for me:

```
$ xtrabackup --backup --stream=tar --target-dir=/tmp/dumpidump | gzip - | split -d --bytes=500MB - /tmp/dumpidump/backup.tar.gz
```
In fact, [all methods listed in the documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Migrating.ExtMySQL.html#AuroraMySQL.Migrating.ExtMySQL.S3) work fine.

After backing up, the backup has to be moved to an S3 bucket:

```
aws s3 cp /tmp/dumpidump/ s3://my-database-dumps/aurora-$(date +%s)/ --recursive
```

Then, one can create an Aurora cluster from that backup in S3:

```
DB_CLUSTER_IDENTIFIER="aurora-$(date +%s)"

aws rds restore-db-cluster-from-s3 \
    --db-cluster-identifier $DB_CLUSTER_IDENTIFIER \
    --master-username admin \
    --master-user-password … \
    --engine aurora-mysql \
    --engine-version 5.7.mysql_aurora.2.11.0 \
    --source-engine mysql \
    --source-engine-version 5.7.41 \
    --s3-bucket-name my-database-dumps \
    --s3-prefix manual/aurora-1676451099 \
    --vpc-security-group-ids sg-... \
    --s3-ingestion-role-arn … \
    --db-subnet-group-name … \
    --db-cluster-parameter-group-name … \

aws rds create-db-instance \
    --db-cluster-identifier $DB_CLUSTER_IDENTIFIER\
    --db-instance-identifier $DB_CLUSTER_IDENTIFIER \
    --db-instance-class db.t4g.medium \
    --engine aurora-mysql
```

Also make sure that the `s3-ingestion-role-arn` has sufficient permissions on the S3 bucket, and if used, on the KMS key.

## Restoring to a plain RDS MySQL instance

During the above debugging process, I also tried out restoring into a plain MySQL RDS (which can be migrated to Aurora later on), but got another error message:

```
 An error occurred (InvalidParameterValue) when calling the RestoreDBInstanceFromS3 operation: You must request the most recent preferred minor version. Please request mysql5.7.38
```

If you get this error, your source engine version is probably TOO NEW. Plain MySQL RDS cannot restore backups created from newer versions than the current default minor version (currently 5.7.38). I tried to create a backup from 5.7.41, but that version was too new. You would have to downgrade the backup source, or migrate to Aurora (which can also handle newer minor versions). IN general, restoring into plain MySQL RDS, [has many more limitations](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Procedural.Importing.html#MySQL.Procedural.Importing.Limitations), e.g. it’s not possible to restore from S3 files which are KMS-encrypted (default S3 encryption works, though).

When googling for the errors listed here, I got nothing useful, so I hope my article might point some folks on the right path.

