---
title: 'Idempotent CloudFormation stack creation/update one-liner with Ansible'
date: 2017-03-17
---

When developing CloudFormation templates, I regularly missed an idempotent one-liner command which does something like "create or update stack N with these parameters", which provides a fast feedback loop.

So here it is with Ansible (and virtualenv for convenience):

```shell
virtualenv venv
source venv/bin/activate
pip install ansible boto
ansible localhost -m cloudformation -a "stack_name=stack_name template=path/to/template region=eu-west-1 template_parameters='template_param1=bar,template_param2=baz'"
```
It will create a new or update an existing CloudFormation stack and wait until the operation has finished. It won't complain if there are no updates to be performed.

PS: [Michael Wittig](https://michaelwittig.info/) [has released a wrapper written in Node](https://github.com/widdix/cfn-create-or-update) for this problem, too!
