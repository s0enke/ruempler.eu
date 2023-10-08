---
title: Zero-downtime upgrade from AWS Aurora 2 (MySQL 5.7) to version 3 (MySQL 8.0) with the CDK and Aurora Blue/Green deployments 
date: 2023-10-08 00:00:00
---

This article demonstrates how to upgrade an CDK (or Cloudformation)-managed AWS Aurora cluster from MySQL 5.7 compatible AWS Aurora 2 to MySQL compatible Aurora 3 engine without any - ok, I lied - one minute of downtime. The process utilizes the [blue/green deployment feature](https://aws.amazon.com/blogs/aws/new-fully-managed-blue-green-deployments-in-amazon-aurora-and-amazon-rds/) of Aurora. It also shows how CDK/CloudFormation stacks are brought back in sync with the upgraded Aurora clusters.

<!--more-->

## Prerequisite: Create a Aurora 2 / MySQL 5.7 cluster

First, for the sake of the example, let's create an Aurora 2.11 cluster including a writer and a reader with AWS CDK:

```typescript
const clusterParameterGroup = new aws_rds.ParameterGroup(this, 'clusterParameterGroup', {
  engine: aws_rds.DatabaseClusterEngine.auroraMysql({
    version: aws_rds.AuroraMysqlEngineVersion.VER_2_11_3
  }),
});
clusterParameterGroup.bindToCluster({}); // create parameter group even if not attached to a cluster

const dbInstanceParameterGroup = new aws_rds.ParameterGroup(this, 'dbInstanceParameterGroup', {
  engine: aws_rds.DatabaseClusterEngine.auroraMysql({
    version: aws_rds.AuroraMysqlEngineVersion.VER_2_11_3
  }),
});
dbInstanceParameterGroup.bindToCluster({}); // create parameter group even if not attached to a cluster

const clusterInstanceProps = {
  instanceType: aws_ec2.InstanceType.of(aws_ec2.InstanceClass.T4G, aws_ec2.InstanceSize.MEDIUM),
  parameterGroup: dbInstanceParameterGroup,
};
const dbCluster = new aws_rds.DatabaseCluster(this, 'dbCluster', {
  engine: aws_rds.DatabaseClusterEngine.auroraMysql({
    version: aws_rds.AuroraMysqlEngineVersion.VER_2_11_3,
  }),
  vpc: vpc,
  vpcSubnets: {
    subnetType: aws_ec2.SubnetType.PUBLIC
  },
  writer: aws_rds.ClusterInstance.provisioned('instance1', clusterInstanceProps),
  readers: [
    aws_rds.ClusterInstance.provisioned('instance2', clusterInstanceProps),
  ],
  parameterGroup: clusterParameterGroup,
});
```
This creates an Aurora cluster with a reader and writer within a VPC (without costly NAT Gateways).

Now once needs to enable the native MySQL binlog replication which will be necessary later on for the AWS Aurora blue/green deployment, since Aurora does not utilize its own native cluster-level replication for blue/green deployments.

```typescript
const clusterParameterGroup = new aws_rds.ParameterGroup(this, 'clusterParameterGroup', {
  engine: aws_rds.DatabaseClusterEngine.auroraMysql({
    version: aws_rds.AuroraMysqlEngineVersion.VER_2_11_3
  }),
  parameters: {
    'binlog_format': 'ROW',
  }
});
```

Enabling the binlog requires a reboot of the cluster. After the reboot, the cluster has binlog replication enabled, and we are almost ready to create a blue/green deployment.

If you have custom parameter groups for the MySQL 5.7 cluster, you will probably also need to create them for the MySQL 8.0 cluster. You can do this with the CDK like this:

```typescript
const clusterParameterGroupMysql8 = new aws_rds.ParameterGroup(this, 'clusterParameterGroupMysql8', {
  engine: aws_rds.DatabaseClusterEngine.auroraMysql({
    version: aws_rds.AuroraMysqlEngineVersion.VER_3_04_0,
  }),
  parameters: {
    // your custom cluster parameters here
  }
});
clusterParameterGroupMysql8.bindToCluster({}); // create parameter group even if not attached to a cluster

const dbInstanceParameterGroupMysql8 = new aws_rds.ParameterGroup(this, 'dbInstanceParameterGroupMysql8', {
  engine: aws_rds.DatabaseClusterEngine.auroraMysql({
    version: aws_rds.AuroraMysqlEngineVersion.VER_3_04_0,
  }),
  parameters: {
    // your custom instance parameters here
  }
});
dbInstanceParameterGroupMysql8.bindToInstance({}); // create parameter group even if not attached to a 
```

Otherwise, you can also use the default Aurora 3 / MySQL 8 parameter groups.

The `bindToCluster` and `bindToInstance` calls are necessary to create the parameter groups even if they are not attached to a cluster or instance. Otherwise, the CDK will not create the parameter groups. This will be the case once we create the blue/green deployment since CDK/CloudFormation won't know about the new cluster or instances yet.

It's also import when removing the cluster or instance, since the parameter groups would be removed as well by the CDK even if the parameter groups are still attached to the old cluster or instances. This will happen when bringing the CDK code / CloudFormation stack back in sync with the actual state in AWS, since the CDK would otherwise try to remove the supposedly unused parameter groups, which are actually still in use by the old MysQL 5.7 / Aurora 2 cluster.

## Create a blue/green deployment

Now we are ready to create an Aurora blue/green deployment. As of writing the CDK does not support creating Aurora blue/green deployments, so we need to use the AWS CLI (or AWS Console) for this:

```bash
$ aws rds create-blue-green-deployment \
  --blue-green-deployment-name blue-green-mysql8 \
  --source <cluster arn> \
  --target-db-cluster-parameter-group-name <> \
  --target-db-parameter-group-name aurorabluegreencdkstack-dbinstanceparametergroup980eefbb-vj6mqvc1qvid  \
  --target-engine-version 8.0.mysql_aurora.3.04.0
```

You can find the cluster ARN in the AWS Console. The `--target-db-cluster-parameter-group-name` and `--target-db-parameter-group-name` parameters are the names of the new parameter groups which we created above. The `--target-engine-version` is the new engine version which we want to upgrade to. In this case it's `8.0.mysql_aurora.3.04.0`.

Now the AWS RDS Console should look like this for the cluster: 
![Blue/Green deployment created](/2023/10/08/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green-aws-console.png)

RDS is now creating a copy of the existing cluster first (the "green" cluster), then it will upgrade the green cluster to MySQL 8.0 / Aurora 3 with using an [in-place upgrade process](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.MajorVersionUpgrade.html#AuroraMySQL.Updates.MajorVersionUpgrade.2to3). During the entire process, the original cluster is still available and untouched. 
After the upgrade, the green cluster starts replicating changes from the blue cluster.

Once finished, the AWS Console should look like this:

![B;ue/Green switch ready](/2023/10/08/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green-aws-console-finished.png)

## Switching over the blue/green deployment to the Aurora 3 / MySQL 8 cluster

Before switching over, you should check whether the "green" cluster instances are in sync with their parameter groups or need an additional reboot. If they need a reboot, you should reboot them now before switching so you avoid a downtime after the switch.

Now you can switch over the blue/green deployment to the new Aurora 3 / MySQL 8 cluster:

```bash
$ aws rds switchover-blue-green-deployment --blue-green-deployment-identifier bgd-9vcnifsubtzqrkub
```

After approx. one minute, the switch over should be finished:

![Blue/green deployment switchover finished](/2023/10/08/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green-aws-console-switchover-finished.png)

## Switching the CDK code to the new Aurora 3 / MySQL 8 cluster

Now the state in CDK / CloudFormation has drifted from the actual state in AWS, so we need to update the CDK code to the new Aurora 3 / MySQL 8 cluster. Otherwise, CDK and the CloudFormation stack still "think" that the old Aurora 2 / MySQL 5.7 cluster is the current one. You can also verify this by starting a CloudFormation Drift Detection which should look like this:

![Drift Detection](/2023/10/08/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green/drift.png)

So let's update the CDK code to the new Aurora 3 / MySQL 8 cluster:

```typescript
const clusterInstanceProps = {
  instanceType: aws_ec2.InstanceType.of(aws_ec2.InstanceClass.T4G, aws_ec2.InstanceSize.MEDIUM),
  parameterGroup: dbInstanceParameterGroupMysql8, // use the new parameter group
};
const dbCluster = new aws_rds.DatabaseCluster(this, 'dbCluster', {
  engine: aws_rds.DatabaseClusterEngine.auroraMysql({
    version: aws_rds.AuroraMysqlEngineVersion.VER_3_04_0, // use the new engine version
  }),
  vpc: vpc,
  vpcSubnets: {
    subnetType: aws_ec2.SubnetType.PUBLIC
  },
  writer: aws_rds.ClusterInstance.provisioned('instance1', clusterInstanceProps),
  readers: [
    aws_rds.ClusterInstance.provisioned('instance2', clusterInstanceProps),
  ],
  parameterGroup: clusterParameterGroupMysql8, // use the new parameter group
});
```
Another `cdk deploy` now brings the CDK code and CloudFormation state in sync with the actual state in AWS. 

To avoid surprises, I recommend to check via a CloudFormation changeset first, that the cluster is not subject to replacement. You can also check this via a `cdk diff`, that the cluster resource shows no replacement:

```
$ cdk diff
Stack AuroraBlueGreenCdkStack
Resources
[~] AWS::RDS::DBCluster dbCluster dbCluster23D869C0 
 ├─ [~] DBClusterParameterGroupName
 │   └─ [~] .Ref:
 │       ├─ [-] parameterGroupFE655916
 │       └─ [+] clusterParameterGroupMysql80BF91F41
 └─ [~] EngineVersion
     ├─ [-] 5.7.mysql_aurora.2.11.3
     └─ [+] 8.0.mysql_aurora.3.04.0
[~] AWS::RDS::DBInstance dbCluster/instance1 dbClusterinstance19C2E8020 may be replaced
 └─ [~] DBParameterGroupName (may cause replacement)
     └─ [~] .Ref:
         ├─ [-] dbInstanceParameterGroup980EEFBB
         └─ [+] dbInstanceParameterGroupMysql8CDABFA3F
[~] AWS::RDS::DBInstance dbCluster/instance2 dbClusterinstance20D0141DC may be replaced
 └─ [~] DBParameterGroupName (may cause replacement)
     └─ [~] .Ref:
         ├─ [-] dbInstanceParameterGroup980EEFBB
         └─ [+] dbInstanceParameterGroupMysql8CDABFA3F

✨  Number of stacks with differences: 1
```

If you use custom parameter groups for the Aurora 3 / MySQL 8 cluster, and the parameter groups are defined in the same stack as the cluster, CloudFormation will probably trigger a reboot. You can avoid this by defining parameter groups in another CloudFormation stack. I have documented a workaround in [Minimizing downtimes when managing AWS RDS parameter groups with CloudFormation
](https://ruempler.eu/2023/04/13/mininizing-downtime-cloudformation-rds-parameter-groups/).

Once you are confident that everything is running fine, you can now manually delete the now unused "blue" Aurora cluster (suffix `old1` by AWS). After removing the cluster, you can also remove the now unused 5.7 / Aurora 2 parameter groups from your CDK code.

## Notes

- The CDK API with a `writer` and `readers` is unfortunately misleading, since which instance is a writer or reader is a dynamic runtime property and can change any time. There is a [GitHub discussion around this issue](https://github.com/aws/aws-cdk/issues/26726).
- Beware if you are using Backtrack on the blue cluster: As of writing, Aurora Blue/Green deployment will silently disable it on the green cluster, and there is currently no way to turn on Backtrack for existing clusters. So AWS maneuvered us into a dead end. But there is a way out: Open a support ticket and ask them to enable Backtrack for the green cluster. They will have to open an internal support case with the Aurora team. At least with an enterprise support plan it worked for me :)

## Summary

This article showed how to upgrade an CDK-managed AWS Aurora cluster from MySQL 5.7 compatible AWS Aurora 2 to MySQL compatible Aurora 3 engine with minimal downtime utilizing the blue/green deployment feature of Aurora.
