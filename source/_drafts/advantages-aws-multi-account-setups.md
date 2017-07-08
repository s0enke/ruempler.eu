---
title: "Advantages of AWS Multi-Account setups"
---

When we begin doing some things in AWS, we usually start with one AWS account and put our resources in it, and things can become a mess very fast. This article should give you an overview why you should switch to a using multi-account strategy very soon if you decided to run production workloads on AWS.

## "Blast radius" reduction

The main reason for separating workloads into several distinct AWS accounts is to [limit the so called blast radius](https://www.slideshare.net/AmazonWebServices/aws-reinvent-2016-reduce-your-blast-radius-by-using-multiple-aws-accounts-per-region-and-service-sec304). It means to contain issues, problems or leaks **by design** so that only one portion of the infrastructure is affected when things go wrong and to prevent them from leaking / cascading into other accounts.

AWS accounts are logically separated: No AWS account or resource in it can access resources of other AWS accounts by default. Cross-account access is possible, but it has always to be granted in an explicit way, e.g. by granting permissions through IAM or other mechanism specific to an AWS service. 

### AWS throttles API requests per AWS account

[AWS throttles API access on an per-account basis](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/query-api-troubleshooting.html#api-request-rate): So for example imagine some script of Team A is e.g. hammering the EC2 API could result in another Team B's production deployment to fail, if they are in the same AWS account. Finding the cause could be hard or even impossible for Team B. They might even see themselves forced to add retries/backoff to their deployment scripts which further increases load and even more throttling! And last but not least, it adds [accidental complexity](http://codebetter.com/markneedham/2010/03/18/essential-and-accidental-complexity/) to their software.

Additionally there are also [service and resource limits per AWS account](http://docs.aws.amazon.com/general/latest/gr/aws_service_limits.html). Some of them can be raised, some can't. the probability of running into one of these limits declines if you distribute your AWS resources  across several AWS Accounts.
### Security

Maybe you remember [the startup Code Spaces](https://threatpost.com/hacker-puts-hosting-service-code-spaces-out-of-business/106761/) which had all their resources in one AWS account including backup: they got hacked and entirely vaporized within 12 hours.
I would argue that this scenario would have happened less likely if their backups resided in another AWS account.

### Environment separation 

Typed in `DROP DATABASE` into the wrong shell? Oops, production is gone! That's actually common story, you might as well remember [this GitHub outage](https://github.com/blog/744-today-s-outage) (not directly related to AWS, but similar contributing factors).

Consider separating e.g. test, staging amd production into own AWS accounts to reduce the blast radius.

### IAM is complicated

IAM is not very easy to grasp and even today there seems to be no easy way to follow the [Principle of Least Privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) in IAM, I'd say that [Managed Policies](http://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html) are a good start, but too often I see myself falling back to assign `AdministratorAccess`. So we often tend to give away too many permissions to e.g. IAM roles or IAM users.

By separating workloads into their own AWS Accounts, we once again reduce the blast radius of too broad permissions to one AWA account - **by design**.

## Map AWS Accounts to your organizational structure

My personal interpretation of the very often quoted Conway's Law is that organizational structure defines the technical systems it designs and generates. A company is a complex social system which is formed of human relationships and communications. As a result, companies usually try to break down the organization into smaller autonomous subsystems. A subsystem could be a organizational unit/team or a product/project team. Thus, providing each subsystem their AWS account seems to be natural. It allows teams to make autonomous decisions within their AWS account and reduce communication overhead across subsystem borders as well as dependencies on other subsystems. 

The [folks from scout24 though issued a warning on mapping AWS accounts 1:1 to **teams**](http://techblog.scout24.com/2017/02/how-many-aws-accounts-should-a-company-have/):

> The actual sizing and assignment of accounts is not strictly regulated. We started creating accounts for teams but quickly found out that assigning accounts per product or group of related services makes more sense. Teams are changing members and missions but products are stable until being discontinued. Also products can be clearly related to business units. Therefore we introduced the rule of thumb: 1 business unit == 1 + n accounts. It allows to clearly identify each account with one business unit and gives users freedom to organize resources at will.

I can definitely fully sign that statement as I have seen it many times that teams are splitting and merging or are constantly getting reorganised. This is especially true in companies who think they are agile and try to fix deeper systemic problems by constantly reorganizing people and teams, ignorant of Conway's Law or their technical constraints / heritage. 

Exploring your company's [Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html) might be another method to find the right sizing and slicing.

**Never slice AWS accounts by teams or org units - but rather by Bounded Context, product name, purpose or capability.**

### Ownership and billing

Another advantage is the clarity of ownership when using multiple accounts. This can be enormously important in organizations which are in the transition from a classical dedicated ops team to a "You built it, you run it." model: If let's say a dev team spawns a resource into *their AWS account*, it's *their* resource. It's *their* database, it's *their*  whatever. No throw-over-the-wall. They can move fast, they don't have to mess around with or wait for other teams, but they are also **more directly connected to the consequences of their actions**. 

Also it makes billing really simple since costs are transparently mapped to the particular AWS accounts (Consolidated Billing), so you get a detailed bill per e.g. business function, environment or whatever you defined as dimensions for your AWS accounts. Again, a direct feedback loop. In contrast, think of a big messy AWS account with one huge bill. That might simply reinforce the still prevailing believe in many enterprises that IT is just [a cost centre](https://en.wikipedia.org/wiki/Cost_centre_(business)).

Side note: Yes you could also use [Cost Allocation Tags](http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html) for making ownership and costs transparent, but tagging has some limitations:

 1. Tagging is not consistent across AWS services and resources: Some support tags, some don't. 
 2. You need to force people to use tagging and/or build systems that check for correct tags etc. This process has to be maintained (e.g. initialized, communicated, trained, enforced, re-communicated, reinforced, and so on).

### Right from the beginning

When I created my first corporate AWS account back in 2010, neither me nor my colleagues weren't aware of all the multi-account advantages mentioned here. This was one contributing factor resulting in one big shared AWS account across teams that really became an entangled mess over time. And believe me: "We'll split up the account later, when we have more time / are earning money / are more people" is simply not going to happen! So please don't make our mistake again! Create more AWS accounts!

My current favorite is to slice AWS accounts in two dimensions:

 - Dimension one: Business function/capability/product/project/Bounded Context (not teams/departments, see above!)
 - Dimension two: Environment (e.g.test, staging, prod)
 
This sounds like a lot of initial complexity, but I think it's really worth it in the long term for the mentioned reasons. Creating AWS Accounts is free and:

### It's getting easier with AWS Organizations

[AWS Organizations](https://aws.amazon.com/organizations/) does not only simplify the creation of new AWS accounts (it has been a pain in the ass before!), it also helps to govern who can do what: You can structure the AWS accounts you own into a organizational tree and apply policies to specific sub-trees. For example, you could deny the use of a particular service org-wide, for a organizational unit or a single account.

In one of my next articles, I am going to bring some light into the drawbacks of having many AWS accounts, but also how to mitigate these drawbacks with automated account provisioning and governance, so stay tuned!

## References

- http://techblog.scout24.com/2017/02/how-many-aws-accounts-should-a-company-have/
- https://aws.amazon.com/answers/account-management/aws-multi-account-security-strategy/
- https://www.linkedin.com/pulse/managing-multiple-accounts-aws-jake-burns
- https://www.slideshare.net/AmazonWebServices/aws-account-best-practices
- https://aws.amazon.com/blogs/aws/new-cross-account-access-in-the-aws-management-console/