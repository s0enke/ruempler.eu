---
title: "Dead man's switch with AWS CloudWatch: Freshness-Alerting for Backups and Co"
date: 2017-06-26 12:00:00
---

A recent challenge for one of the teams I am currently involved was to find a way in AWS CloudWatch:

1. To monitor if the metric breaches a specified threshold.
2. To monitor whether a particular metric has been sent to CloudWatch within a specified interval.

While the first one is pretty much standard CloudWatch functionality, the latter is a bit more tricky. In the Nagios/Icinga world it's called "freshness". You could also call it special case of a "[Dead man's switch](https://en.wikipedia.org/wiki/Dead_man%27s_switch)" for periodic tasks / cronjobs.

 So for example in our case we wanted to have monitored and alerted whether a backup job runs once per day.
 
So here is what we did (CloudFormation snippet below):

- Set the check period to the interval during the metric should be sent. E.g. `86400` if the metric should is supposed to be sent every day. This instructs CloudWatch to check once per day.
- Set evaluation periods to `1`: We want to get alerted immediately when there is no data written or the threshold has been breached. 
- And now the important one: We have to treat missing data as `breaching`, so that,if there has been no entry within the evaluation period then the alarm gets triggered.

Example in CloudFormation syntax:
```yaml
  HealthCheckAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      Period: 86400
      EvaluationPeriods: 1
      TreatMissingData: breaching
      ...
```

## References

 - [Cloudwatch freshness checks like Nagios passive checks](https://serverfault.com/questions/743190/cloudwatch-freshness-checks-like-nagios-passive-checks)