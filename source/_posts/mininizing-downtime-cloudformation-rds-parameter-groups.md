---
date: "2023-04-13 12:00"
title: "Minimizing downtimes when managing AWS RDS parameter groups with CloudFormation"
---

When managing RDS parameter groups with CloudFormation, you might have noticed that changes to a DB cluster or instance parameter group will always cause a reboot of the RDS instance. This happens when the RDS cluster or instance parameter group and the RDS instance are managed by the same CloudFormation stack.

<!--more-->


Always rebooting RDS instances is a good way to ensure that the instances are always in sync with the parameter groups, but it can also be a problem if you want to minimize downtime.

It's also not necessary in many cases to reboot the RDS instance when changing the parameter group. Only static parameters need to a reboot to take effect. Dynamic parameters can be applied without a reboot.

**But Cloudformation triggers a `RebootDBInstance` action when the parameter group changes when the RDS cluster or instance parameter group and the RDS instance are managed by the same CloudFormation stack.**

**The solution (or: workaround) is simple, but might not be obvious at first: Split the RDS cluster or instance parameter group and the RDS cluster/instance definitions into two separate CloudFormation stacks:** One stack for the RDS parameter groups and one for the RDS clusters and instances. You can reference between stacks with CloudFormation exports and `ImportValue`.

This way, you can change the RDS parameter group values without rebooting the RDS instance. Dynamic parameters will be applied immediately, static parameter changes will be applied after the next reboot / the next maintenance window.

Note: I have tested this behavior with AWS CloudFormation and RDS Aurora 2.11. Other RDS engines might behave differently, but I assume that the behavior is the same, and CloudFormation makes no difference between RDS engines.