---
title: "Advantages and Challenges of AWS Multi(/Cross) Account setup"
---

This is the start of a series about multi account

 1. Advantages and Challenges of AWS Multi(/Cross) Account setup
 1. Practical implementation of a permission concept in AWS Multi(/Cross) Account setups
 1. Automating the AWS account base setup with CloudFormation
 1. Advanced: Cross-account CodePipelines
 1. Advanced: Single-sign-on

## Advantages of multiple AWS accounts

### "Blast radius" reduction

The main reason for separating workloads into several distinct AWS accounts is to limit the so called blast radius. It means to contain issues, problems or leaks so that only a portion of the infrastructure is affected when things go wrong and prevent them from leaking / cascading into other accounts.

- **API calls get throttled/limited**: [AWS throttles API access on an per-account basis](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/query-api-troubleshooting.html#api-request-rate). So for example when some script of Team A is e.g. hammering the EC2 API, which could result in Team B's production deployment to fail. Finding the cause be hard or even impossible for Team B. They might even see themselves forced to add retries/backoff to their deployment scripts which further increases load (and complexity in their software).
- **Security**: I is less likely that a breach into one AWS account leaks into other other accounts.
- **Environment separation**: Typed in `DROP DATABASE` into the wrong shell? Oops, production is gone! That's actually common story, you might as well remember the [GitHub outage](https://github.com/blog/744-today-s-outage). Or maybe [the startup Code Spaces](https://threatpost.com/hacker-puts-hosting-service-code-spaces-out-of-business/106761/) which had all their resources in one AWS account including backup: they got hacked and entirely vaporized within 12 hours.

### Map AWS Accounts to your organizational structure

My personal interpretation of the very often quoted Conway's Law is that organizational structure defines the technical systems it designs and generates. A company is a complex social system which is formed of human relationships and communications. As a result, companies usually try to break down the organization into smaller autonomous subsystems. A subsystem could be a organizational unit/team or a product/project team. Thus, providing each subsystem their AWS account seems to be natural. It allows teams to make autonomous decisions within their AWS account and reduce communication overhead across subsystem borders as well as dependencies on other subsystems.    

### Ownership and billing

Another advantage is the clarity of ownership when using multiple accounts. Yes you could use tagging for that, but tagging has some limitations:

 - You need to force people to use tagging and/or build systems that check for correct tags etc. This process has to be initated, trained, enforced, reinforced, etc etc.
 - Tagging is not consistent accross AWS services: some support tags, some don't. 

Also it makes billing really simple as costs are transparently mapped to the different AWS accounts (Consolidated Billing). 

### 

### It get's easier with AWS Organizations

AWS Organizations does not only simplify the creation of new AWS accounts (it has been a pain in the ass before!), it also helps to govern who can do what: You can structure the AWS accounts you own into a organizational tree and apply policies to specific sub-trees. For example, you could deny the use of a particular service org-wide, for a organizational unit or a single account.
 
## Challenges with multiple accounts

### Logging
 
- CloudTrail
  
### Monitoring / Dashboards