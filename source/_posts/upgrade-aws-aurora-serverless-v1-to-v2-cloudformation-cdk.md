---
title: Upgrade CDK/Cloudformation-managed AWS Aurora Serverless v1 (MySQL 5.7) cluster to Serverless v2 (MySQL 8.0) with minimal downtime
date: 2023-12-31 12:00:00
---

Since [AWS is going to sunset / auto-upgrade Aurora Serverless v1 in December 2024](https://www.reddit.com/r/aws/comments/18sx0i6/aurora_serverless_v1_eol_december_31_2024/), it's time to upgrade to Aurora Serverless v2. Upgrading to Serverless v2 MySQL instances also means upgrading from MySQL 5.7 to MySQL 8.0 under the hood.

This article describes how to adapt the [AWS provided upgrade path](https://aws.amazon.com/blogs/database/upgrade-from-amazon-aurora-serverless-v1-to-v2-with-minimal-downtime/) to work with a CDK / CloudFormation infrastructure-as-code project, which requires a few more steps which I call the "**CDK/CloudFormation retain-and-import dance**".

<!--more-->


The dance needs to be done since a direct modification via CloudFormation would result a replacement of the cluster, since [CloudFormation does not support the modification of the engine mode from `serverless` to `provisioned`](https://github.com/aws-cloudformation/cloudformation-coverage-roadmap/issues/1743).

## Step 1: Unbind the Aurora Serverless v1 cluster from CloudFormation (RETAIN):

First, add `removalPolicy: cdk.RemovalPolicy.RETAIN,` to the Aurora Serverless v1 cluster definition:

```typescript
// Serverless v1 cluster
const dbCluster = new aws_rds.ServerlessCluster(this, 'AuroraCluster', {
  clusterIdentifier: 'aurora-serverless-v1-upgrade',
  engine: aws_rds.DatabaseClusterEngine.AURORA_MYSQL,
  vpc: vpc,
  vpcSubnets: {
    subnetType: aws_ec2.SubnetType.PUBLIC
  },
  parameterGroup: dbClusterParameterGroup,
  scaling: {
    autoPause: cdk.Duration.minutes(5),
    minCapacity: aws_rds.AuroraCapacityUnit.ACU_1,
    maxCapacity: aws_rds.AuroraCapacityUnit.ACU_1,
  },
  removalPolicy: cdk.RemovalPolicy.RETAIN, // add this
});
```
and deploy the change.

Next, comment out the `AuroraCluster` resource, and check with `cdk diff` that the cluster will be retained:

```
Stack aurora-serverless-v1-upgrade
Resources
[-] AWS::RDS::DBSubnetGroup AuroraCluster/Subnets AuroraClusterSubnetsF3E9E6AD orphan
[-] AWS::EC2::SecurityGroup AuroraCluster/SecurityGroup AuroraClusterSecurityGroupD85BF9CB destroy
[-] AWS::SecretsManager::Secret AuroraCluster/Secret AuroraClusterSecret8E4F2BC8 destroy
[-] AWS::SecretsManager::SecretTargetAttachment AuroraCluster/Secret/Attachment AuroraClusterSecretAttachmentDB8032DA destroy
[-] AWS::RDS::DBCluster AuroraCluster AuroraCluster23D869C0 orphan
[+] AWS::RDS::DBParameterGroup dbInstanceParameterGroup dbInstanceParameterGroup980EEFBB 
```
As expected, the `AuroraCluster` will be retained ("orphan").

But, as you can see, the CDK might also have auto-generated CloudFormation resources such as secrets, security groups etc. (depending on your configuration), so there is an additional difficulty that CloudFormation would to delete (or try to delete) these still-in-use resources.

To prevent this, we can create and use these dependecies explicitly, so that they are not auto-generated (and auto-deleted) by the CDK anymore:

```typescript
const dbClusterSecurityGroup = new aws_ec2.SecurityGroup(this, 'dbClusterSecurityGroup', {
  vpc: vpc,
});

const dbClusterSecret = new aws_rds.DatabaseSecret(this, 'dbClusterSecret', {
  username: 'admin',
});

const dbClusterCredentials = aws_rds.Credentials.fromSecret(dbClusterSecret);

// Serverless v1 cluster
const dbCluster = new aws_rds.ServerlessCluster(this, 'AuroraCluster', {
  // ...
  securityGroups: [dbClusterSecurityGroup],
  credentials: dbClusterCredentials,
  // ...
});
```

Now after making all resources explicit, we can ensure with another `cdk diff` that Cloudformation will not try to delete anything the cluster depends on:

```
➜  serverless-v1-upgrade-cdk git:(master) ✗ npx cdk diff  
Stack aurora-serverless-v1-upgrade
Resources
[-] AWS::SecretsManager::SecretTargetAttachment dbClusterSecret/Attachment dbClusterSecretAttachment17AD9723 destroy
[-] AWS::RDS::DBSubnetGroup AuroraCluster/Subnets AuroraClusterSubnetsF3E9E6AD orphan
[-] AWS::RDS::DBCluster AuroraCluster AuroraCluster23D869C0 orphan
```

The SecretTargetAttachment is still there, and it will be deleted by CloudFormation, since it's not referenced anymore by the `AuroraCluster` resource. But it will be recreated in the `import` step later. 

## Step 2: Prepare for upgrade: Change the engine mode to "provisioned"

Now let's follow the AWS blog post and modify the cluster to temporarily become a provisioned cluster:

```
$ aws rds modify-db-cluster \
 --db-cluster-identifier aurora-serverless-v1-upgrade \
 --engine-mode provisioned \
 --allow-engine-mode-change \
 --db-cluster-instance-class db.t4g.medium \ # adapt this to your peak load
 --apply-immediately
```

To cite the original AWS blog post:

> The conversion will take a few minutes, during which time there will be an approximately 30-second failover window while the new provisioned instance is promoted. When the process is complete, your cluster will be converted to a provisioned Aurora database cluster.

Now we need to make the CDK/CloudFormation stack aware again of the changed cluster.  So we add the `AuroraCluster` resource again, and modify it to adapt to the `provisioned` engine mode:

 - The resource type changed from `ServerlessCluster` to `DatabaseCluster`
 - There is a provisioned `writer` instance for now. This will become the serverless writer later.

```typescript
const dbCluster = new aws_rds.DatabaseCluster(this, 'AuroraCluster', {
  clusterIdentifier: 'aurora-serverless-v1-upgrade',
  engine: aws_rds.DatabaseClusterEngine.AURORA_MYSQL,
  vpc: vpc,
  vpcSubnets: {
    subnetType: aws_ec2.SubnetType.PUBLIC
  },
  parameterGroup: dbClusterParameterGroup,
  writer: aws_rds.ClusterInstance.provisioned('writer', {
    instanceIdentifier: 'aurora-serverless-v1-upgrade-instance-1', // has been assigned by RDS
    instanceType: aws_ec2.InstanceType.of(aws_ec2.InstanceClass.T4G, aws_ec2.InstanceSize.MEDIUM),
    publiclyAccessible: false,
  }),
  securityGroups: [dbClusterSecurityGroup],
  credentials: dbClusterCredentials,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});
```
We should now be able to "reattach" the cluster to the CDK/Cloudformation stack with `cdk import`.
```
$ cdk import
aurora-serverless-v1-upgrade
aurora-serverless-v1-upgrade/dbClusterSecret/Attachment/Resource: unsupported resource type AWS::SecretsManager::SecretTargetAttachment, skipping import.
aurora-serverless-v1-upgrade/AuroraCluster/Subnets/Default (AWS::RDS::DBSubnetGroup): enter DBSubnetGroupName (empty to skip): aurora-serverless-v1-upgrade-2-auroraclustersubnetsf3e9e6ad-mcf3lclxho3t
aurora-serverless-v1-upgrade/AuroraCluster/Resource (AWS::RDS::DBCluster): import with DBClusterIdentifier=aurora-serverless-v1-upgrade (yes/no) [default: yes]? yes
aurora-serverless-v1-upgrade/AuroraCluster/writer/Resource (AWS::RDS::DBInstance): import with DBInstanceIdentifier=aurora-serverless-v1-upgrade-instance-1 (yes/no) [default: yes]? 
aurora-serverless-v1-upgrade: importing resources into stack...
aurora-serverless-v1-upgrade: creating CloudFormation changeset...
```

Check with `cdk diff` that the cluster is now imported:

```
$ cdk diff
Stack aurora-serverless-v1-upgrade
Resources
[+] AWS::SecretsManager::SecretTargetAttachment dbClusterSecret/Attachment dbClusterSecretAttachment17AD9723 
```

So let's run a `cdk deploy` to create the missing Secrets Manager attachment.

After that it's a good idea to run a CloudFormation drift detection to make sure that the resources are now in sync with the CDK/CloudFormation stack.

## Step 3: Upgrade the provisioned cluster to Aurora 3 / MySQL 8.0

Now it's time to upgrade the provisioned cluster to Aurora 3 / MySQL 8.0. I have already described how to do this in [Zero-downtime upgrade from AWS Aurora 2 (MySQL 5.7) to version 3 (MySQL 8.0) with the CDK and Aurora Blue/Green deployments
](https://ruempler.eu/2023/10/08/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green/), so please follow the steps there.

## Step 4: Changing the engine mode back to "serverless"

Welcome back. We have now a provisioned cluster with MySQL 8.0. First, let's add a serverless reader "instance" to the cluster which we can promote to the new writer later. This step is necessary to avoid extended downtime, which would happen if we would simply change the type from `provisioned` to `serverless`.

```typescript
const dbCluster = new aws_rds.DatabaseCluster(this, 'AuroraCluster', {
  // ...
  writer: aws_rds.ClusterInstance.provisioned('writer', {
    // still the same
  }),
  readers: [
    aws_rds.ClusterInstance.serverlessV2('serverless', {
      parameterGroup: dbInstanceParameterGroupMysql8,
      publiclyAccessible: false,
    }),
  ],
  // ...
});
```

After deploying this, we can fail-over to the serverless reader, and delete the provisioned writer instance:


```
$ aws rds failover-db-cluster \
 --db-cluster-identifier aurora-serverless-v1-upgrade \
 --target-db-instance-identifier aurora-serverless-v1-upgr-auroraclusterserverlessd-hqyhnw25qwrg # autogenerated by CloudFormation
```

This step introduces drift into the CDK code since the writer is now the reader and vice versa. There is a [GitHub discussion around this topic](https://github.com/aws/aws-cdk/issues/26726). 

Since we want to get rid of the provisioned "writer" instance (which is actually the reader), we first change it to serverless as well:

```typescript
const dbCluster = new aws_rds.DatabaseCluster(this, 'AuroraCluster', {
  // ...
  writer: aws_rds.ClusterInstance.serverlessV2('serverless', {
    instanceIdentifier: 'aurora-serverless-v1-upgrade-instance-1', // has been assigned by RDS
    parameterGroup: dbInstanceParameterGroupMysql8,
    publiclyAccessible: false,
  }),
  readers: [
    // ...
  ],
  // ...
});
```

Then we fail over again to the serverless writer:

```
$ aws rds failover-db-cluster \
 --db-cluster-identifier aurora-serverless-v1-upgrade \
 --target-db-instance-identifier aurora-serverless-v1-upgrade-instance-1
```
That brings CDK and the cluster back in sync, at least for the moment.  Finally, we can delete the reader instance.

## Summary

This article described how to upgrade a CDK/CloudFormation-managed Aurora Serverless v1 cluster to Serverless v2 with minimal downtime. 