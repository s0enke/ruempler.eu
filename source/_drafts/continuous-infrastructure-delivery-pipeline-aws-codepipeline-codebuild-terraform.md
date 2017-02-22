---
title: Serverless Continuous Delivery Pipeline for Infrastructure with AWS CodePipeline, CodeBuild and Terraform 
tags:
---

Goals:
 - serverless pipeline:
 
## "serverless"

## Cloudformation

CloudFormation is generally a good solution to start Infrastructure-as-Code (Iac) projects in AWS, because it offers a low maintenance and easy-to-start solution. On the other hand, it can have some drawbacks based on the use case or the usage level. Here are some points which pop up regularly:

 - AWS-only: CloudFormation has no native support for third-party services. It actually supports [custom resources](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html), but those are usually awkward to write and maintain. I would only use them as last resort.
 - Not all AWS services / features supported: The usual AWS feature release process is that a component team releases a new feature, but the CloudFormation part is missing (the CloudFormation team at AWS is apparently a separate team with its own road map). And since CloudFormation isn’t open source, we cannot add the missing functionality by ourselves. 
- No imports of existing resources: AWS resources created outside of CloudFormation cannot be "imported" into a stack. This would be helpful for example when resources had been created manually earlier before (maybe because CloudFormation did not support them yet).
 
## Terraform to the rescue!

Terraform is an IaC tool from HashiCorp, similar to CloudFormation, but with a broader usage range and greater flexibility than CloudFormation.

Terraform also has several advantages over CloudFormation, among others:

 - Open source: Terraform is open source so you can patch it and send changes upstream to make it better. This is great because anyone can for example add new services or features, or fix bugs. It's actually not uncommon that Terraform is even faster than CloudFormation with implementing new AWS features.
 - It supports a wide range of services, not only AWS: This enables automating bigger ecosystems spanning e.g. multiple clouds or providers. In CloudFormation one would have to fall back to awkward custom resources. A particular use-case is provisioning databases and users of an MySQL database, 
 - Data sources: While CloudFormation has "imports" and some intrinsic functions to lookup values (e.g. from existing resources), Terraform has a wide range of data sources (just have a look at [this impressive list](https://www.terraform.io/docs/providers/aws/d/acm_certificate.html).  
 - Imports: Terraform can [import existing resources](https://www.terraform.io/docs/import/) (if supported by the resources type)!
  
## Downsides of Terraform

TerraForm is no managed service, so the maintenance burden is on the user side. That means we as users have to install, upgrade, maintain, debug it and so on - instead of focusing on building our products.

Another important point is that Terraform uses so called state files to maintain the state of the infrastructure it created. The files are the **holy grail** of Terraform and messing around with them can bring you into serious trouble, e.g. bringing your infrastructure into a undefined state. The user has to come up with a solution how to keep those state files in a synchronized and central space (Luckily Terraform provides [remote state handling](https://www.terraform.io/docs/state/remote/index.html), I will get back to this in a second). CloudFormation also maintains the state of the resources it created, but AWS takes care of it!

Last but not least, Terraform currently does not take care of of locking, so two concurrent Terraform runs could lead to unintended consequences. ([which will change soon](https://github.com/hashicorp/terraform/pull/11686)).
 
## Putting it all together

So how can we leverage the described advantages of Terraform while still minimizing operational overhead and costs?

### Serverless delivery pipelines

First of all, we should use a Continuous Delivery Pipeline: Every change in the source code triggers a run of the pipeline consisting of several steps, e.g. running tests and finally applying / deploying the changes. AWS offers a service called CodePipeline to create and run these pipelines. It's a fully managed service, no servers or containers to manage (a.a.a "serverless").

CodePipeline also inherently takes care of the Terraform state file locking as it does not allow a single pipeline steps to run multiple times concurrently.

### Executing Terraform

Remember, we want to create a safe environment to execute Terraform, which is consistent can be audited (so NOT your workstation!!). 


 - pipeline as code
 - serverless pipeline: avoid managing own build infrastructure
 - use codebuild to execute terraform
 - use s3 state provider
 
 CodePipeline ensures that there are no concurrent runs of the pipeline which could
  
## Trying it out

 - One button template
 - 
 
## Stronger together

The real power of both TerraForm and Cloudformation comes to light when we combine them. We can actually use best of both worlds. This will be a topic of a coming blog post.
  
## Caveat:

CodeBuild makes use of container credentials
 
```
[Container] 2017/02/19 21:23:07 AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/b9e0883b-62ff-4d03-84a3-af9213dd08e0
[Container] 2017/02/19 21:23:07 AWS_DEFAULT_REGION=us-east-1
[Container] 2017/02/19 21:23:07 CODEBUILD_AGENT_ENV_AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/b9e0883b-62ff-4d03-84a3-af9213dd08e0
```

https://github.com/hashicorp/terraform/issues/8746

## References

https://blog.gruntwork.io/how-to-manage-terraform-state-28f5697e68fa?gi=9769367dd11