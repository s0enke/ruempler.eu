---
title: AWS Continuous Infrastructure Delivery with CodePipeline and CloudFormation: How to pass Stack Parameters
date: 2017-04-10
---

When deploying CloudFormation stacks in a "Continuous Delivery" manner with CodePipeline, one might encounter the challenge to pass many parameters from the CloudFormation stack describing the pipeline to another stack describing the infrastructure to be deployed (in this example a stack named `application`).
 
Consider a CloudFormation snippet describing CodePipeline which deploys another CloudFormation stack:

```yaml
# pipeline.yaml
...
Resources:
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      ...
      Stages:
      ...
      - Name: Application
        Actions:
        - Name: DeployApplication
          ActionTypeId:
            Category: Deploy
            Owner: AWS
            Provider: CloudFormation
            Version: 1
          Configuration:
            ActionMode: CREATE_UPDATE
            StackName: application
            TemplatePath: Source::application.yaml
```

Now when you want to pass parameters from the pipeline stack to the `application` stack, you could use the ParameterOverrides option offered by the [CodePipeline CloudFormation integration](https://aws.amazon.com/blogs/aws/codepipeline-update-build-continuous-delivery-workflows-for-cloudformation-stacks/), which might look like this:

```yaml
# pipeline.yaml
...
- Name: DeployApplication
  ...
  Configuration:
    StackName: application
    TemplatePath: Source::application.yaml
    ParameterOverrides: '{"ApplicationParameterA": "foo", "ApplicationParameterB": "bar"}'
```

This would pass the parameters `ApplicationParameterA` and `ApplicationParameterB` to the `application` CloudFormation stack. For reference this is how the `application` stack could look like:

```yaml
# application.yaml
---
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  ApplicationParameterA:
    Type: String
  ApplicationParameterB:
    Type: String

Resources:
...
```
## Alternative way of parameter passing with Template Configurations

Injecting parameters from the pipeline stack to the application stack can become awkward with the `ParametersOverrides` method. Especially when there are many parameters and they are passed into the pipeline stack as parameters as well, the pipeline template could look like this:
```yaml
# pipeline.yaml
---
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  ApplicationParameterA:
    Type: String
  ApplicationParameterB:
    Type: String

Resources:
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Stages:
        ...
        Actions:
        - Name: DeployApplication
          ...
          Configuration:
            ...
            TemplatePath: Source::application.yaml
              ParameterOverrides: !Sub '{"ApplicationParameterA": "${ApplicationParameterA}", "ApplicationParameterB": "${ApplicationParameterB}"}'
```

An alternative way is to place a so called [template configuration](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/continuous-delivery-codepipeline-cfn-artifacts.html#w1ab2c13c19c17) into the same artifact which contains the `application.yaml` template, and reference it via the `TemplateConfiguration`:
```yaml
# pipeline.yaml
...
- Name: DeployApplication
  ...
  Configuration:
    ...
    TemplatePath: Source::application.yaml
    ParameterOverrides: '{"ApplicationParameterA": "foo", "ApplicationParameterB": "bar"}'
    TemplateConfiguration: Source::template_configuration.json
```

In our case, the `template_configuration.json` file would look like this:

```json
{
  "Parameters" : {
    "ApplicationParameterA" : "foo",
    "ApplicationParameterB" : "bar"
  }
}
```

This might be much nicer to handle and maintain depending on your setup.

Btw you can also use the `TemplateConfiguration` to [protect your resources from being deleted or replaces with Stack policies](/2017/03/28/aws-codepipeline-cloudformation-stack-policy-prevent-replacement-resources/).

