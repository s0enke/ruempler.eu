---
date: "2022-10-19 12:00"
title: "Serverless - a new model to run backend software"
---

For me, serverless has become the default model of developing backend applications.
However, a meeting a few weeks ago was a reminder for me that not the entire world is embracing serverless applications already: I had the task to design an “Energy Data Importer”, an application that collects time series data from different locations, sanitizes and transforms them and stores them in a database for later lookup. So I went ahead and [sketched the application with AWS serverless building blocks and patterns](https://github.com/s0enke/serverless-aws-timeseries-injector-query-architecture/).

<!--more-->

I also created a [prototype with the CDK](https://github.com/s0enke/serverless-aws-timeseries-injector-query-architecture/) with a working integration/E2E test which included authorization with Cognito Userpools, data upload with S3 presigned URL, event-driven Lambda which transforms the data with Python pandas and the aws-datawranger, and writes the data into an Amazon Timestream time series database. Overall, the proof of concept infrastructure and application code had 200 lines of code. And of course with all the advantages which come mostly for free with serverless applications: built-in resilience, high availability, security, elasticity/auto-scaling, cost-effectivity (no pay for idle), near-zero total cost of ownership, and so on. And also with the advantages of IAC: repeatable deployments etc.

I presented the architecture to three people that apparently had never seen serverless architectures before, and they seemed somehow overwhelmed and skeptical. The main three concerns/remarks I remember were:

- “But where is the code? Where can I click through the code?”
- “All the complexity is now in the infrastructure, not in my code.”
- “This is an extreme example!”

Let’s go through them one by one:

## “But where is the code? Where can I click through the code?”

Serverless applications have by definition usually less code, since much self-written custom logic or libraries is replaced by managed services. Having less custom code or libraries results in less “clickable” or scrollable code in the IDE. That might make developers feel uncomfortable, since it feels like losing control: They cannot look up the exact behavior anymore. And that’s true, you have to trust your cloud provider that their documentation is comprehensive and up-to-date, and the software and APIs behave as described.

One simple example of this case is defining HTTP routing logic in the application (e.g. via annotations) vs. defining it in the infrastructure, e.g. via load balancer or API gateway rules. In e.g. Spring Boot or Express.JS, developers can look into the routing logic of the open source framework. They cannot click through code anymore if the routing logic is moved to the infrastructure, such as a load balancer or API gateway.

It can also be more difficult to integration-test serverless applications since they have to leave the framework-land, and the integration testing becomes more end-to-end, because testing now also involves the infrastructure such as load balancers.

On the other hand, one could argue, why needs a developer to click through glue code like routes etc.in the first place. Why are they concerned with this stuff and not with working on the main business logic?

Not writing and being able to scroll through that many indirections, abstractions, factories, etc.  - but still having a working application - might reveal to developers that they might have been overengineering the past and/or overestimated the importance of their “craft”. Serverless architectures with less code often make the unimportance of owned code and libraries more visible, and that might be a bitter pill to swallow for some developers.

## “All the complexity is now in the infrastructure, not in my code!”

Since serverless applications are mostly the combination of managed services, much of the “complexity” is now moved to (or reduced by) the infrastructure, and the combination of managed services.

Business logic and infrastructure logic have been mixed together for a long time.
Applications cannot live with their infrastructure, may it be databases, queues, load balancers, storage, message buses, or email services. For example, a pub/sub-rule whether a subscriber gets certain business events IS business logic AND also “infrastructure code”!

Used to configure XML files or annotations to configure your app? That’s also business logic as “infrastructure as/from code”. But it might feel different whether the rules are in the application code repository, or in the infrastructure code repository (many businesses have separate code repositories for applications and infrastructure).

Serverless and IaC tools like CDK make the fact more visible that application and infrastructure code belong together, and that business logic is also part of the infrastructure complexity.

Using this as an argument against serverless feels a bit like “shooting the messenger” or blaming it as the bearer of “bad” news that there is no working software without infrastructure.

Moving complexity into the infrastructure, and not in the own code, is actually usually a very good thing, since code is a liability and not an asset: Not owning the entire lifecycle of software and libraries (install, maintain, debug, keep up-to-date, deploy, monitor/instrumentation) frees companies from undifferentiated heavy lifting and helps them keep focus on customers and business.

A co-worker once noted: It’s like the “conversation of energy principle”, but for complexity: The complexity is still there, but on the other side of the (cloud provider) API.

## “This is an extreme example!”

This comment was related to the amount of usage of managed services and libraries which resulted in about 200 lines of code for the entire proof of concept including application AND infrastructure code + 60 lines for the integration test. While the Proof of concept did not contain the entire application with all requirements, it showcased a good amount of the application.

With serverless, infrastructure and “application code” become one and this melange contains the business logic. The use of managed services reduces the amount of own/owned code and libraries.

Less code and no borders between application and infrastructure code can scare people, since it is an entirely new way of developing and running modern applications in the cloud to them. If done properly, it also has to result in shifting of roles, responsibilities, skills - and thus lead to re-organizations.

One reason why technologies such as VMWare, Kubernetes etc. are so popular is that organizations and people don’t like change / to be changed. These bridging technologies might bring a “cloudy” feeling. But if most of the existing knowledge and behaviors stay the same, no real change is introduced.

Which closes the circle from me back to my research from 2017 to 2019 regarding [“Serverless vs. Organizations” and Organizational (Un)-Learning](https://www.slideshare.net/AWSAktuell/serverless-vs-developers-the-real-crash).

