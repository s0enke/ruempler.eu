---
title: Extend AWS CloudFormation with Custom Resources and Terraform
date: 2017-09-12 12:00:00
tags:
---

When applying the Infrastructure-as-Code feature, the AWS developer usually feels challenged with the tool choice which often boils down to AWS CloudFormation vs. Terraform. The unique advantages of both approaches are widely documented, so I won't repeat them here. Instead I want to show a possible way to combine the best of both worlds.

For my latest use-case I first started with Terraform because I knew there are some AWS resources it supports which CloudFormation doesn't (Lambda@Edge and Origin Access Control).

But some time later I realized that did not want to sacrifice a unique CloudFormation feature: **One-button deployment / installation**, because I want my CloudFormation templates easily usable, in this particular case even for non tech-savvy users.

So I thought why not execute Terraform from CloudFormation? The reasoning behind is this: Usually - when faced with the situation of resource type not supported by CloudFormation - one would use a CloudFormation custom resource and (re-)implement the creation, update and deletion of the resource with custom code (usually a Lambda function). This approach smells to me like reinventing the wheel, though, if Terraform supports this particular resource. So I started an experiment to execute Terraform code in CloudFormation. 

As mentioned, CloudFormation provides Custom Resources which the developer can use to support arbitrary custom items. The other side of the coin is that the developer has to handle everything by themselves.

## Anatomy of a Custom Resource
Here is an example of what a CloudFormation Custom Resource Lambda function could look like:
```yaml
Resources:
  TerraFormExecuteFunction:
    Type: AWS::Lambda::Function
    Properties:
      Code:
        ZipFile: |
          import cfnresponse

          def handler(event, context):
              print("Received event: " + json.dumps(event, indent=2))

              terraform_working_directory = '/tmp/terraform_files'

              if event['RequestType'] == 'Delete':
                  # todo: implement delete
                  cfnresponse.send(event=event, context=context, responseStatus=cfnresponse.SUCCESS, responseData={}, physicalResourceId=event['LogicalResourceId'])
                  return

              try:
                # todo: implement create/update
                response_data = {
                  'foo': 'bar',
                } 
                cfnresponse.send(event=event, context=context, responseStatus=cfnresponse.SUCCESS, responseData=response_data, physicalResourceId=event['LogicalResourceId'])

              except subprocess.CalledProcessError as exc:
                print("Status : FAILED", exc.returncode, exc.output)
                # TODO: add response data
                cfnresponse.send(event=event, context=context, responseStatus=cfnresponse.FAILED, responseData={}, physicalResourceId=event['LogicalResourceId'])
                return
```
The `cfnresponse` package is provided by the Lambda runtime, so we can use it directly as inline code in CloudFormation without the need of a dedicated build and deployment process of the Lambda Zip-file.



Try it out: Deploy a CloudFront and Lambda@Edge with CloudFormation. Launch stack.

## Possible next steps

 - Bundle/vendor Terraform binaries into the Lambda function to speed up execution and enhance stability
 - Check out `AWS::Include` in order to reuse the Terraform function Custom Resource

