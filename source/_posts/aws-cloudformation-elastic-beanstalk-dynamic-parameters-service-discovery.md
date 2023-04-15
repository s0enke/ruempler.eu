---
title: '"Service Discovery" with AWS Elastic Beanstalk and CloudFormation'
date: 2017-04-09 12:00:00
---
## How to dynamically pass environment variables to Elastic Beanstalk.

Elastic Beanstalk is a great AWS service for managed application hosting. For me personally, it's the Heroku of AWS: Developers can concentrate on developing their application while AWS takes care of all the heavy lifting of scaling, deployment, runtime updates, monitoring, logging etcpp.

<!--more-->

But running applications usually means not only using plain application servers the code runs on, but also databases, caches and so on. And AWS offers many services like ElastiCache or RDS for databases, which should usually preferred in order to have lower maintenance overhead.

So, how do you connect Elastic Beanstalk and other AWS services? For example, your application needs to know the database endpoint of an RDS database in order to use it.

"Well, create the RDS via the AWS console, copy the endpoint and pass it as an environment variable to Elastic Beanstalk", some might say.

Others might say: Please don't hardcode such data like endpoint host names, use a service discovery framework, or DNS and use that to look up the name.

Yes, manually clicking services in the AWS console and hardcoding configuration is usually a bad thing(tm), because it violates "Infrastructure as Code": Manual processes are error-prone, and you'll loose documentation through codification, traceability and reproducibility of the setup. 

But using DNS or any other service discovery for a relatively simple setup? Looks like a oversized solution for me, especially if the main driver for Elastic Beanstalk was the reduction of maintenance burden and complexity.

### The solution: CloudFormation

Luckily, there is a simple solution to that problem: CloudFormation. With CloudFormation, we can describe our Elastic Beanstalk application and the other AWS resources it consumes in one template. We can also inject e.g. endpoints of those AWS resources created to the Elastic Beanstalk environment.
 
Let's look at a sample CloudFormation template - step by step (I assume you are familiar with CloudFormation and Elastic Beanstalk itself).
 
 First, let's describe an Elastic Beanstalk application with one environment:
 
```yaml
...
Resources:
  Application:
    Type: AWS::ElasticBeanstalk::Application
    Properties:
      Description: !Ref ApplicationDescription
  ApplicationEnv:
    Type: AWS::ElasticBeanstalk::Environment
    Properties:
      ApplicationName: !Ref Application
      SolutionStackName: 64bit Amazon Linux 2016.09 v2.5.2 running Docker 1.12.6
```

Ok, nothing special so far, let's add a RDS database:

```yaml
  DB:
    Type: AWS::RDS::DBInstance
    Properties:
      ...
```

CloudFormation allows it to get the endpoint of the database with the `GetAtt` function. To get the endpoint of the `DB` database, the following code can be used:

```
!GetAtt DB.Endpoint.Address
```
And CloudFormation can also pass environment variables to Elastic Beanstalk environments, so let's combine those two capabilities:
```yaml
  ApplicationEnv:
    Type: AWS::ElasticBeanstalk::Environment
    Properties:
      ApplicationName: !Ref Application
      ...
      OptionSettings:
      - Namespace: aws:elasticbeanstalk:application:environment
        OptionName: DATABASE_HOST
        Value: !GetAtt DB.Endpoint.Address
      
```

Et voila, the database endpoint hostname is now passed as an environment variable (`DATABASE_HOST`) to the Elastic Beanstalk environment.
You can add as many environment variables as you like. They are even updated if you change their value (Cloudformation would trigger an Elastic Beanstalk enviroment update is this case).