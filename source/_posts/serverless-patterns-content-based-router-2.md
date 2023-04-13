---
date: "2022-09-21 12:00"
title: "Content-Based Router (Serverless Enterprise Integrations Patterns on AWS)"
---

This article is part of the [serverless enterprise pattern collection](/serverless-enterprise-integration-patterns-aws/) which adapts [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging) to serverless architectures and principles.

> Use a Content-Based Router to route each message to the correct recipient based on message content.

AWS has two services which allow content-based message filtering: EventBridge and SNS.  EventBridge is the newer one - with more features, event sources, and destinations.SNS (TODO: Link), 

On the other hand, SNS still is the default notification service for many older AWS services which do not support yet EventBridge.  And in contrast to EventBridge SNS supports FIFO messaging, which can be a very important characteristic of event-based systems (e.g. Event Sourcing).

Content-based routers route messages from 


> In more sophisticated integration scenarios, the Content-Based Router can take on the form of a configurable rules engine that computes the destination channel based on a set of configurable rules.
> 
> 
