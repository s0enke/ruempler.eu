---
title: "CodePipeline and CloudFormation with a stack policy to prevent REPLACEMENTs of resources"
date: 2017-03-29
---

Some operations in CloudFormation trigger a [`REPLACEMENT`](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html) of resources which can have unintended and catastrophic consequences, e.g. an RDS instance being replaced (which means that the current database will be **deleted** by CloudFormation after a new one has been created).

While CloudFormation [does support deletion policies natively](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html) which prevent the deletion of resources, there is no simple way to do this for `REPLACEMENT`s as of writing this. 

When using CodePipeline in combination with CloudFormation to deploy infrastructure changes in an automated Continuous Delivery manner, one should have protection from accidental deletions even more mind. This blog post shows how to use [CloudFormation Stack Policies](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html) to protect critical resources from being replaced.

Let's start with the CodePipeline (expressed as CloudFormation) piece which deploys a database stack called `db` (I assume you are confident with CloudFormation and CodePipeline):

```yaml
Pipeline:
Type: AWS::CodePipeline::Pipeline
Properties:
  ...
  Stages:
  - Name: Source
    ...
  - Name: DB
    Actions:
    - Name: DeployDB
      ActionTypeId:
        Category: Deploy
        Owner: AWS
        Provider: CloudFormation
        Version: 1
      Configuration:
        ActionMode: CREATE_UPDATE
        RoleArn: !GetAtt CloudFormationRole.Arn
        StackName: db
        TemplatePath: Source::db.yaml
        TemplateConfiguration: Source::db_stack_update_policy.json
      InputArtifacts:
      - Name: Source
      RunOrder: 1
```

The important part is the [`TemplateConfiguration`](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/continuous-delivery-codepipeline-cfn-artifacts.html#w1ab2c13c19c17) parameter which tells CloudFormation to look for a configuration at this particular path in the `Source` artifact. In this case `db_stack_update_policy.json`.

`db_stack_update_policy.json` looks like this:
```yaml
{
  "StackPolicy" : {
    "Statement" : [
      {
        "Effect" : "Allow",
        "Action" : "Update:*",
        "Principal": "*",
        "Resource" : "*"
      },
      {
        "Effect" : "Deny",
        "Action" : "Update:Replace",
        "Principal": "*",
        "Resource" : "LogicalResourceId/DB"
      }
    ]
  }
}
```
While the first statement allows all updates to all resources in the `db` stack, the second will deny operations which would result in a `REPLACEMENT` of the resource with the logical id `DB` in this stack.

A CloudFormation stack update of `db` would fail with an error message like `Action denied by stack policy: Statement [#1] does not allow [Update:Replace] for resource [LogicalResourceId/DB]`.
