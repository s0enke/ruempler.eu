---
date: "2023-03-26 12:00"
title: "Automated MySQL RDS/Aurora initialization with CDK, Fargate and the MySQL CLI"
---

A recurring task when provisioning databases is to initialize them with  initial users, stored procedures, and/or tables.

In the CDK world, there is currently no native support for this, but there are some workarounds. AWS has a [blog post](https://aws.amazon.com/blogs/database/automating-database-initialization-with-aws-cloud-development-kit/) about this, but it's using a Lambda function, and a bunch of custom code which seemed like too much operational overhead for me. 

<!--more-->

In this post, I’ll show how to initialize a MySQL RDS/Aurora database with CDK, a Fargate container, and simple `mysql` commands.

First, let's create a database cluster:

```typescript
const dbCluster = new aws_rds.DatabaseCluster(this, 'dbCluster', {
    ...
    instanceProps: {
        vpc: vpc,
        securityGroups: [
            dbSecurityGroup,
        ],
        ...
    },
});
```

So far, nothing special. Now, let's create a one-off container which is running a `mysql` client and connects to the database. We'll use the [official MySQL Docker image](https://hub.docker.com/_/mysql) and run a simple `mysql` command to create a database user in the Aurora cluster:

```typescript 
const initContainer = new aws_ecs.FargateTaskDefinition(this, 'initContainer');
taskDefinition.addContainer('initContainer', {
    image: aws_ecs.ContainerImage.fromRegistry('public.ecr.aws/docker/library/mysql:5.7'),
    essential: false,
    secrets: {
        MYSQL_PWD: aws_ecs.Secret.fromSecretsManager(dbCluster.secret!, 'password'),
    },
    command: [
        'bash',
        '-c',
        `echo "
        CREATE DATABASE IF NOT EXISTS ${props.dbName};
        # put more stuff here
        GRANT ALL PRIVILEGES ON ${props.dbName}.* TO '${props.dbUser}'@'%';
        " | mysql -h ${dbCluster.endpoint.hostname} -u admin -f;
        `
    ],
});

const fargateService = new aws_ecs.FargateService(this, 'fargateService', {
    cluster: cluster, // omitted for brevity
    taskDefinition: initContainer,
    ...
});
```

The `command` section is where the magic happens. We're using the `mysql` client command to create a database and a user. There is no mysql daemon started in the container.

The mysql admin password is stored in Secrets Manager, so we can retrieve it using the `aws_ecs.Secret.fromSecretsManager` method. By setting it as a secret environment variable, it will be automatically injected into the container. The `mysql` client will pick up the environment variable and use it to authenticate to the database.

The `essential` flag is set to `false`, so the container will not be restarted after it exits. It's a one-off task. But note that the container is executed on each ECS service update (e.g. when a new task definition is deployed). So it's best to make sure that the provided mysql commands are idempotent.

Otherwise, you can use the `-f` flag is used to force the execution of the MySQL commands, even if some of them fail.

Other containers that depend on the database initialization can now depend via ECS container dependencies on the `initContainer`:

```typescript
containerDependingOnInitContainer.addContainerDependencies({
    container: initContainer,
    condition: aws_ecs.ContainerDependencyCondition.SUCCESS,
});
```
This will start the `containerDependingOnInitContainer` when the  database initialization has been successful.
 
Please also note that at least one container in the task definition must be marked as `essential: true`. Otherwise, the service refuses to start.

In this post I showed how to initialize a MySQL RDS/Aurora database with CDK, a Fargate container, and simple `mysql` commands. This is a simple workaround until CDK supports database initialization natively.