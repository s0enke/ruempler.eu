---
date: "2022-09-21 12:00"
title: "🤔 The serverless test"
---

There is a lot of buzz and marketing around the term "serverless". This article aims to dismantle what's serverless and what not - with practical examples.

## Definition

So first, we need a definition of serverless. A good start might be Wikipedia:

> Serverless computing is a cloud computing execution model in which the cloud provider allocates machine resources on demand, taking care of the servers on behalf of their customers. [...] However, developers of serverless applications are not concerned with capacity planning, configuration, management, maintenance, fault tolerance, or scaling of containers, VMs, or physical servers. [...]  When an app is not in use, there are no computing resources allocated to the app. Pricing is based on the actual amount of resources consumed by an application.[1] It can be a form of utility computing.

This definition is very tightly coupled to compute, while a backend application usually consists of many more components like storage, queues, databases, event busses, load balancers, etc.

[Paul Johnston, one of the fathers of ServerlessDays, has a more broad definition](https://pauldjohnston.medium.com/a-simple-definition-of-serverless-8492adfb175a), which doesn't only apply to compute, but to software solutions in general.

> A Serverless solution is one that costs you nothing to run if nobody is using it (excluding data storage).

The inherent essence of this definition is that every building block of a serverless solution **MUST** scale to zero (or isn't billed) if not used. Data storage is an exception since it's usually not wanted that data are wiped automatically if not used, unless it's a cache.

So let's have a look at use cases or Public cloud services and see if they meet the serverless definition above:

| Product / use case | is it serverless? | why? |  
| ---- | ---- | ---- |
| Amazon Elastic Kubernetes Service (EKS) | ❌  | $40 base fee / month
| Azure AKS | ✅ | No base fee. But also possibliy useless without worker nodes |
| Google Kubernetes Engine (GKE)| ❌ | $74 base fee / month
| Knative | 🤔 | Depends: <ol><li>is there a base fee for the Kubernetes Cluster (as with EKS and GKE) <li>can the scheduler auto-scale nodes to 0?
| Serverless Aurora v1 | ✅ | Can [scale to zero](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v1.how-it-works.html#aurora-serverless.how-it-works.auto-scaling)
| Serverless Aurora v2 | ❌ | Despite it's name, it's not serverless, since [it cannot scale to zero and costs $43/month for idle](https://www.lastweekinaws.com/blog/no-aws-aurora-serverless-v2-is-not-serverless/).
| Kinesis Data Streams | ❌ | Minimum 1 shard, $29/month
| Kinesis Data Streams On Demand | ✅| On-demand, scales to zero
| AWS, SQS, SNS, EventBridge | ✅ | On-demand, pay-per-use
| DynamoDB Standard | ❌ | Pricing based on provisioned capacity units |
| DynamoDB on demand | ✅ | Per request pricing |
| AWS Elastic Beanstalk | ❌ | The ELB/ALB/NLB load balancer has a base fee | 
| AWS App Runner | ✅ | App Runner scales to zero
| AWS ECS+Fargate | ✅ | While Fargate can scale to zero, a E/A/NLB load balancer is most likely used.
| Google App Engine Standard | ✅ | scales to 0
| Google App Engine Flexible | ❌ | Despite its name, it's not that flexible and needs VMs
| Akamai Edge Functions | 🤔 | Need to contact sales

I will add more example as they cross my attention.

Ok, at this point you might be like "So what?". In another article, we will have a look at why scaling to zero is actually a very important and good architectural characteristic. 