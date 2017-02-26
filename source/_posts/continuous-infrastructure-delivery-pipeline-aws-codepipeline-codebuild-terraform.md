---
title: >-
  Continuous Infrastructure Delivery Pipeline with AWS CodePipeline, CodeBuild
  and Terraform
date: 2017-02-26 14:27:59
tags:
---


While I am a heavy CloudFormation user, sometimes its limitations bug me, especially when I look a little bit jealously at the Terraform users who have all those cool features. In this article, I want to explore and showcase how to build a low-maintenance [Continuous Delivery](https://martinfowler.com/books/continuousDelivery.html) pipeline for Terraform, by using only AWS components. 
 
## CloudFormation

CloudFormation is a good solution to start Infrastructure-as-Code (Iac) projects in AWS, because it offers a low-maintenance and easy-to-start solution. On the other hand, it can have some drawbacks based on the use case or the usage level. Here are some points which pop up regularly:

 - AWS-only: CloudFormation has no native support for third-party services. It actually supports [custom resources](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html), but those are usually awkward to write and maintain. I would only use them as a last resort.
 - Not all AWS services/features supported: The usual AWS feature release process is that a component team releases a new feature, but the CloudFormation part is missing (the CloudFormation team at AWS is apparently a separate team with its own roadmap). And since CloudFormation isn’t open source, we cannot add the missing functionality by ourselves. 
- No imports of existing resources: AWS resources created outside of CloudFormation cannot be "imported" into a stack. This would be helpful for example when resources had been set up manually earlier before (maybe because CloudFormation did not support them yet).
 
## Terraform to the rescue!

Terraform is an IaC tool from HashiCorp, similar to CloudFormation, but with a broader usage range and greater flexibility than CloudFormation.

Terraform has several advantages over CloudFormation, here are some of them:

 - **Open source**: Terraform is open source so you can patch it and send changes upstream to make it better. This is great because anyone can, for example, add new services or features, or fix bugs. It's not uncommon that Terraform is even faster than CloudFormation with implementing new AWS features.
 - **Supports a broad range of services, not only AWS**: This enables automating bigger ecosystems spanning e.g. multiple clouds or providers. In CloudFormation one would have to fall back to awkward custom resources. A particular use-case is provisioning databases and users of a MySQL database, 
 - **Data sources**: While CloudFormation has only "[imports](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html)" and some intrinsic functions to lookup values (e.g. from existing resources) Terraform provides a wide range of data sources (just have a look at [this impressive list](https://www.terraform.io/docs/providers/aws/d/acm_certificate.html).
 - **Imports**: Terraform can [import existing resources](https://www.terraform.io/docs/import/) (if supported by the resources type)! As mentioned, this becomes handy when working with a brownfield infrastructure, e.g. manually created resources.
  
## (Some) Downsides of Terraform

TerraForm is no managed service, so the maintenance burden is on the user side. That means we as users have to install, upgrade, maintain, debug it and so on (instead of focusing on building our own products).

Another important point is that Terraform uses "state files" to maintain the state of the infrastructure it created. The files are the **holy grail** of Terraform and messing around with them can bring you into serious trouble, e.g. bringing your infrastructure into an undefined state. The user has to come up with a solution how to keep those state files in a synchronized and central location (Luckily Terraform provides [remote state handling](https://www.terraform.io/docs/state/remote/index.html), I will get back to this in a second). CloudFormation actually also maintains the state of the resources it created, but AWS takes care of state storage!

Last but not least, Terraform currently does not take care of locking, so two concurrent Terraform runs could lead to unintended consequences. ([which will change soon](https://github.com/hashicorp/terraform/pull/11686)).
 
## Putting it all together

So how can we leverage the described advantages of Terraform while still minimizing its operational overhead and costs?

### Serverless delivery pipelines

First of all, we should use a Continuous Delivery Pipeline: Every change in the source code triggers a run of the pipeline consisting of several steps, e.g. running tests and finally applying/deploying the changes. AWS offers a service called [CodePipeline](https://aws.amazon.com/documentation/codepipeline/) to create and run these pipelines. It's a fully managed service, no servers or containers to manage (a.k.a "serverless").

### Executing Terraform

Remember, we want to create a safe environment to execute Terraform, which is consistent and which can be audited (so NOT your workstation!!).
  
To execute Terraform, we are going to use AWS [CodeBuild](https://aws.amazon.com/codebuild/), which can be called as an action within a CodePipeline. The CodePipeline will inherently take care of the Terraform state file locking as it does not allow a single action to run multiple times concurrently. Like CodePipeline, CodeBuild itself is fully managed. And it follows a pay-by-use model (you pay for each minute of build resources consumed).

CodeBuild is instructed by a YAML configuration, similar to e.g. TravisCI ([I explored some more details in an earlier post](/2016/12/19/aws-codebuild-the-missing-link-for-deployment-pipelines-in-aws/)). Here is how a Terraform execution could look like:

```yaml
  version: 0.1
  phases:
    install:
      commands:
        - yum -y install jq
        - curl 169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI | jq 'to_entries | [ .[] | select(.key | (contains("Expiration") or contains("RoleArn"))  | not) ] |  map(if .key == "AccessKeyId" then . + {"key":"AWS_ACCESS_KEY_ID"} else . end) | map(if .key == "SecretAccessKey" then . + {"key":"AWS_SECRET_ACCESS_KEY"} else . end) | map(if .key == "Token" then . + {"key":"AWS_SESSION_TOKEN"} else . end) | map("export \(.key)=\(.value)") | .[]' -r > /tmp/aws_cred_export.txt # work around https://github.com/hashicorp/terraform/issues/8746
        - cd /tmp && curl -o terraform.zip https://releases.hashicorp.com/terraform/${TerraformVersion}/terraform_${TerraformVersion}_linux_amd64.zip && echo "${TerraformSha256} terraform.zip" | sha256sum -c --quiet && unzip terraform.zip && mv terraform /usr/bin
    build:
      commands:
        - source /tmp/aws_cred_export.txt && terraform remote config -backend=s3 -backend-config="bucket=${TerraformStateBucket}" -backend-config="key=terraform.tfstate"
        - source /tmp/aws_cred_export.txt && terraform apply
```

First, in the `install` phase, the tool `jq` is installed to be used for a little workaround I had to wrote to get the AWS credentials from the metadata service, as [Terraform does not yet support this yet](https://github.com/hashicorp/terraform/issues/8746). After retrieving the AWS credentials for later usage, Terraform is downloaded, checksum'd and installed (they have no Linux repositories).
 
In the build phase, first the Terraform state file location is set up. As mentioned earlier, it's possible to use [S3 buckets as a state file location](https://www.terraform.io/docs/state/remote/s3.html), so we are going to tell Terraform to store it there.

You may have noticed the `source /tmp/aws_cred_export.txt` command. This simply takes care of setting the AWS credentials environment variables before executing Terraform. It's necessary because CodeBuild does not retain environment variables set in previous commands.

Last, but not least, `terraform apply` is called which will take all `.tf` files and converge the infrastructure against this description.
 
### Pipeline as Code

The delivery pipeline used as an example in this article [is available as an AWS CloudFormation template](https://github.com/s0enke/cloudformation-templates/blob/master/templates/pipeline-terraform.yml), which means that it is codified and reproducible. Yes, that also means that CloudFormation is used to generate a delivery pipeline which will, in turn, call Terraform. And that we did not have to touch any servers, VMs or containers. 

You can try out the CloudFormation one-button template here:

[![Launch Stack](https://raw.githubusercontent.com/s0enke/cloudformation-templates/master/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=codepipeline-terraform-sample&templateURL=https://s3.amazonaws.com/ruempler-cloudformation-templates-prod/pipeline-terraform.yml)

You need a GitHub repository containing one or more `.tf` files, which will in turn get executed by the pipeline and Terraform.

Once the CloudFormation stack has been created, the CodePipeline will run initially:
  
![CodePipeline screenshot](pipeline.png)

The `InvokeTerraformAction` will call CodeBuild, which looks like this:

![CodeBuild log output screenshot](codebuild.png)

## Stronger together

The real power of both TerraForm and CloudFormation comes to light when we combine them, as we can actually use best of both worlds. This will be a topic of a coming blog post.
  
## Summary 

This article showed how AWS CodePipeline and CodeBuild can be used to execute Terraform runs in a Continuous Delivery spirit, while still minimizing operational overhead and costs. A [CloudFormation template is provided](https://github.com/s0enke/cloudformation-templates/blob/master/templates/pipeline-terraform.yml) to ease the set up of such a pipeline. It can be used as a starting point for own TerraForm projects.

## References

https://blog.gruntwork.io/how-to-manage-terraform-state-28f5697e68fa?gi=9769367dd11
