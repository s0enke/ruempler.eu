---
title: "Advantages of AWS Multi-Account setups"
---

This is the start of a series about multi account

 1. Advantages and Challenges of AWS Multi(/Cross) Account setup
 1. Practical implementation of a permission concept in AWS Multi(/Cross) Account setups
 1. Automating the AWS account base setup with CloudFormation
 1. Advanced: Cross-account CodePipelines
 1. Advanced: Single-sign-on

## Advantages of multiple AWS accounts

AWS accounts are logically separated. No AWS account can access resources of other AWS accounts by default. Cross-account access has always to be enabled in an explicit way, e.g. by granting permissions through IAM or other mechanism specific to an AWS service. 


### "Blast radius" reduction

The main reason for separating workloads into several distinct AWS accounts is to limit the so called blast radius. It means to contain issues, problems or leaks so that only a portion of the infrastructure is affected when things go wrong and prevent them from leaking / cascading into other accounts.

- **API calls get throttled/limited**: [AWS throttles API access on an per-account basis](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/query-api-troubleshooting.html#api-request-rate). So for example imagine some script of Team A is e.g. hammering the EC2 API could result in another Team B's production deployment to fail, if they are in the same AWS account. Finding the cause could be hard or even impossible for Team B. They might even see themselves forced to add retries/backoff to their deployment scripts which further increases load and even more throttling! And last but not least it adds complexity in their software.
- **Security**: It is less likely that a breach into one AWS account leaks into other other accounts. Maybe you remember [the startup Code Spaces](https://threatpost.com/hacker-puts-hosting-service-code-spaces-out-of-business/106761/) which had all their resources in one AWS account including backup: they got hacked and entirely vaporized within 12 hours.
- **Environment separation**: Typed in `DROP DATABASE` into the wrong shell? Oops, production is gone! That's actually common story, you might as well remember [this GitHub outage](https://github.com/blog/744-today-s-outage) (not directly related to AWS, but similar contributing factors).

### Map AWS Accounts to your organizational structure

My personal interpretation of the very often quoted Conway's Law is that organizational structure defines the technical systems it designs and generates. A company is a complex social system which is formed of human relationships and communications. As a result, companies usually try to break down the organization into smaller autonomous subsystems. A subsystem could be a organizational unit/team or a product/project team. Thus, providing each subsystem their AWS account seems to be natural. It allows teams to make autonomous decisions within their AWS account and reduce communication overhead across subsystem borders as well as dependencies on other subsystems. 

The [folks from scout24 though issued a warning on mapping AWS accounts 1:1 to **teams**](http://techblog.scout24.com/2017/02/how-many-aws-accounts-should-a-company-have/):

> The actual sizing and assignment of accounts is not strictly regulated. We started creating accounts for teams but quickly found out that assigning accounts per product or group of related services makes more sense. Teams are changing members and missions but products are stable until being discontinued. Also products can be clearly related to business units. Therefore we introduced the rule of thumb: 1 business unit == 1 + n accounts. It allows to clearly identify each account with one business unit and gives users freedom to organize resources at will.

I can definitely fully sign that statement as I have many times seen teams splitting and merging or getting reorganised. This is especially true in companies who think they are agile and try to fix problems by reorganizaing people and teams, ignorant of Conways Lay or their Bug Balls of mud. 

Exploring your company's [Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html) might be another method to find the right sizing and slicing.

### Ownership and billing

Another advantage is the clarity of ownership when using multiple accounts. This can be enormously important in organizations which are in the transition from having a classical dedicated ops team to a "You built it, you run it." model: If say a dev team spawns a resource in *their AWS account*, it's *their* resource, it's *their* database, it's *their*  whatever. They can move fast, they don't have to mess around with or wait for other teams, but they are also more directly connected to the consequences of their actions.

Also it makes billing really simple as costs are transparently mapped to the particular AWS accounts (Consolidated Billing), so you get a detailed bill per e.g. team, department, environment or whatever you defined as dimensions for your AWS accounts. Again, a direct feedback loop. In contrast think of a big messy AWS account with one huge bill. That might simply reinforce the still prevailing believe in many enterprises that IT is just [a cost centre](https://en.wikipedia.org/wiki/Cost_centre_(business)).

Sidenote: Yes you could also use Cost Allocation Tags for making ownership and costs transparent, but tagging has some limitations:

 1. Tagging is not consistent across AWS services and resources: Some support tags, some don't. 
 2. You need to force people to use tagging and/or build systems that check for correct tags etc. This process has to be initated (initialized, commnicated,trained, enforced, re-communicated, reinforced, and so on.

### Right from the beginning

When I create my first corporate AWS account in 2010, neither I nor my collagues wasn't aware of all these multi-account advantages. This resulted in sharing one big AWS account across teams which realy got am entangled mess. And believe me: "We'll split up the account later, when we have more time / are earning money / are more people" is simply not going to happen. So please don't make that mistake again. Create more AWS accounts!

### It get's easier with AWS Organizations

AWS Organizations does not only simplify the creation of new AWS accounts (it has been a pain in the ass before!), it also helps to govern who can do what: You can structure the AWS accounts you own into a organizational tree and apply policies to specific sub-trees. For example, you could deny the use of a particular service org-wide, for a organizational unit or a single account.

## References

- http://techblog.scout24.com/2017/02/how-many-aws-accounts-should-a-company-have/
- https://aws.amazon.com/answers/account-management/aws-multi-account-security-strategy/
- https://www.linkedin.com/pulse/managing-multiple-accounts-aws-jake-burns
- https://www.slideshare.net/AmazonWebServices/aws-account-best-practices
- https://aws.amazon.com/blogs/aws/new-cross-account-access-in-the-aws-management-console/
