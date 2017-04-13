---
title: AWS Multi Account setup best practises
---

There is information scattered all over the Web containing information and "Do's and Dont's" about AWS and Multi Account setup. This blog post summarizes my research and personal experiences in AWS projects (I am a freelance consultant, by the way!)
   
## Why avoid a single account?

Usually, with AWS, you start with a single AWS account which contains all your resources, e.g. one VPC, some EC2 instances and a bunch of S3 buckets. Next your coworkers want access. As you are agile, access is granted. As nobody has the time to understand and setup IAM, everbody is just granted AdministratorAccess or PowerUserAccess 

You end up with a big ball of mud.

Amazon has the simple rule of two-pizza-teams: If you can't feed a team with 2 pizzas anymore, it is probably too big and should be split up into smaller teams. In my opionoin, this rule should be applied to AWS accounts as well: If one team cannot oversee what's happening in their AWS account, then there probably are ..]:
 



## How to schneiden.
     
   
What is true for teams, should also be true for your AWS accounts. 

 - How should we structure our AWS account?
 - Should we have several AWS accounts at all?

## Should we have several AWS accounts at all?

YES.

- You can have as many AWS accounts as you like.
- AWS is launching Organizations which will make Multi-Account setups easier to maintain and to govern.
- see immoscout article


In order to have a generic base setup for all the AWS accounts, provide some base stacks:

 - VPC
 - VPN
 - Roles
 - CloudTrail/Logging? (if desired)

Use CloudFormation for automation.
Use CodePipeline for automated centralized updates of the stacks.



IAM Cross accont

http://docs.aws.amazon.com/AmazonS3/latest/dev/example-bucket-policies.html
https://aws.amazon.com/blogs/security/enable-a-new-feature-in-the-aws-management-console-cross-account-access/

infrastructure account
cross acount access for CLI


simplify and automate global rollout of changes through many accounts


remove default VPC

## CodePipeline as "Central State Enforcer"

- many patterns to keep cloudformation stacks up to date
- often manual runs from developer machines
- locking
- works on my machine (e.g. different versions)
- history and auditing

## Base Stack



## Region Stack


## Caveats

 - pipeline self update will the stop the current execution, workaround is a dummy push



http://blog.gardeviance.org/2015/04/the-only-structure-youll-ever-need.html