---
date: "2023-03-26 12:00"
title: "Why "real serverless scale-to-zero and no-pay-for-idle are important"
---

Having defined what is serverless and what not in the last article, this time let's have a look at one particular characteristic of serverless systems: they can scale to zero also known as: you don't pay for idling resources. So let's discuss why this is actually very important.

## Reason 1: Identical Dev/Test/Pre-Prod/Prod infrastructure

Serverless solutions are easy and inexpensive to create and destroy. This leads to huge benefits:

- Identical environments minimize the "works on my machine" or "works on my staging" scenarios where developers tested their code against diverging stubs/mocks in the code or fake environments like localstack. Additionally, avoiding stubs and fakes also reduces development and maintenance time. 
- Thousands of environment copies containing the entire application consisting of databases, queues, storage, messaging etc. can be spun up - without incurring idle costs and without sharing anything between environments. This allows "cattle" thinking for environments as opposed to often seen static test/qa/staging "pet" stages.

In contrast, imagine 100 concurrent environments with the base cost of e.g. one EKS Kubernetes ($0.10/h) and one ALB Load Balancer ($0.027/h). That's ($0.10/h + $0.027/h) * 100 = $12.7/h or $304.80/day or $9144/month - just for idling!

Now the usual way to reduce costs when using "serverful"/pay-for-idle components is to introduce sharing them across environments to save costs. We will get to the implications later.

## Reason 2: Serverless architectures can be better aligned to team structures ("Conways Law")

Teams owning serverless architectures can be completely independent of other teams. In AWS - with a proper multi-account strategy - teams can manage own their own AWS accounts without the need shared components.

No shared components also enable experimentation-friendly environments: teams can spin up their own environments and test new ideas without the need to ask for permission or to wait for a shared environment to be available.

## Reason 3: Serverless architecture raise cost awareness ("FinOps")

Not having any shared infrastructure with shared costs (e.g. a platform based on Kubernetes/EKS) leads to higher cost awareness and exposure, since there are no fixed infrastructure costs which cannot be allocated to one product.

Shared infrastructure usually leads to  obscure fixed costs which will then be taken for granted. In germany we even have a word for these costs: "[EDA ("eh da") Kosten](https://de.wikipedia.org/wiki/EDA-Kosten)". 

## Sharing underlying resources to allegedly save money

As mentioned earlier, the EKS and ALB load balancers could be shared between environments to reduce the costs. Oh yes, they definitely can! And it's actually what I've seen in many companies!

And then often some "fascinating" accidental complexity happens:

1) You loose the team independence to an extent. If the shared infrastructure breaks or in under maintenance, product development is interrupted. More non-value-adding communication/synchronisation between teams is needed. 
1) Someone has to maintain the shared infrastructure. Did a very small virtual platform team just get implicitly founded? This work is not adding any value to the company, it's undifferentiated heavy lifting.
1) At least in AWS, to maintain team boundaries through AWS accounts, cross AWS account access from a central EKS AWS account to team AWS accounts would introduce more complexity and thus more fragility.
1) Allocating costs in shared resources is not possible out of the box - at least in AWS. How do you know which container or load balancer route/request belongs to which team? So you get base costs not allocated to any product/team. Like in the old VM/bare-metal days again. 

So in short: You loose most the advantages of a serverless architecture when introducing shared infrastructure. You even most likely increased the total cost of ownership because the shared infrastructure has to be maintained by a human with a salary.


## How to create serverless freedom while still being compliant?

## But serverless architecture get too expensive 

