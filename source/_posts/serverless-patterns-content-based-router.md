---
date: "2022-09-21 12:00"
title: "Content-Based Router (Serverless Enterprise Integrations Patterns on AWS)"
---

This article is part of the [serverless enterprise pattern collection](/serverless-enterprise-integration-patterns-aws/) which adapts [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging) to serverless architectures and principles.

> Use a Content-Based Router to route each message to the correct recipient based on message content.

AWS has two services which allow content-based message filtering: EventBridge and SNS. 

EventBridge is the newer service - with more features, event sources, and destinations (TODO: Link). EventBridge supports content-based routing through [Event Rules](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-rules.html).

On the other hand, SNS still is the default notification service for many older AWS services which do not support yet EventBridge. And in contrast to EventBridge SNS supports FIFO messaging, which can be a very important characteristic of event-based systems (e.g. Event Sourcing). SNS supports content-based routing through [SNS content filters](https://docs.aws.amazon.com/sns/latest/dg/sns-message-filtering.html).

## Example: EventBridge Content Based Routing (AWS CDK)

The examples show content based routing in EventBridge based on the value of the `detail-type` field, but many more rules are possible. In this example, the `gadget` messages get routed to a Lambda function as handler, the `widget` messages get routed to an SQS queue.

### Plain CDK Example
```typescript
import {
  App,
  aws_events,
  aws_events_targets,
  aws_lambda,
  aws_sqs,
  Stack,
} from "aws-cdk-lib";

const app = new App();

const stack = new Stack(app, "ContentBasedMessageRouter");

new aws_events.Rule(stack, "GadgetRule", {
  eventPattern: {
    detailType: ["gadget"],
  },
}).addTarget(
  new aws_events_targets.LambdaFunction(
    new aws_lambda.Function(app, "GadgetHandler", {
      runtime: aws_lambda.Runtime.NODEJS_16_X,
      handler: "index.handler",
      code: aws_lambda.Code.fromInline(
        "exports.handler = function(event, context) { console.log(event.id); }"
      ),
    })
  )
);

new aws_events.Rule(stack, "WidgetRule", {
  eventPattern: {
    detailType: ["widget"],
  },
}).addTarget(
  new aws_events_targets.SqsQueue(new aws_sqs.Queue(stack, "WidgetHandler"))
);
```

### Functionless Example

```typescript
import { App, Stack } from "aws-cdk-lib";
import { Function, EventBus, Queue } from "functionless";

const app = new App();

const stack = new Stack(app, "ContentBasedMessageRouter");

const defaultBus = EventBus.default(stack);
defaultBus
  .when("gadget", (event) => event["detail-type"] === "gadget")
  .pipe(
    new Function(stack, "GadgetInventoryHandler", async (event) => {
      console.log(event.id);
    })
  );
defaultBus
  .when("widget", (event) => event["detail-type"] === "widget")
  .pipe(new Queue(stack, "WidgetInventoryHandler"));
```

## Example: Content Based Routing with SNS FIFO (CDK)

SNS has the ability to deliver message with FIFO semantics. The examples show content based routing in SNS using [subscription filters](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_sns.SubscriptionFilter.html) based on the value of the `source` message field. In this example, the `gadget` messages get routed to a Lambda function as handler, the `widget` messages get routed to an SQS queue. 

```typescript
import {
  App,
  aws_lambda,
  aws_sns,
  aws_sns_subscriptions,
  aws_sqs,
  Stack,
} from "aws-cdk-lib";

const app = new App();

const stack = new Stack(app, "ContentBasedMessageRouterFifoSns");

const fifoRouter = new aws_sns.Topic(stack, "fifoRouter", {
  fifo: true,
});

const gadgetHandlerSubscription = new aws_sns_subscriptions.LambdaSubscription(
  new aws_lambda.Function(app, "GadgetHandler", {
    runtime: aws_lambda.Runtime.NODEJS_16_X,
    handler: "index.handler",
    code: aws_lambda.Code.fromInline(
      "exports.handler = function(event, context) { console.log(event.id); }"
    ),
  }),
  {
    filterPolicy: {
      source: aws_sns.SubscriptionFilter.stringFilter({
        allowlist: ["widget"],
      }),
    },
  }
);
fifoRouter.addSubscription(gadgetHandlerSubscription);

const widgetHandlerSubscription = new aws_sns_subscriptions.SqsSubscription(
  new aws_sqs.Queue(stack, "WidgetHandler", {
    fifo: true,
  }),
  {
    filterPolicy: {
      source: aws_sns.SubscriptionFilter.stringFilter({
        allowlist: ["widget"],
      }),
    },
  }
);
fifoRouter.addSubscription(widgetHandlerSubscription);
```
