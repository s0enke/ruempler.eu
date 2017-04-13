---
title: How to set up Federated Single Sign On to AWS using Web Identities (OpenID Connect)
date: 2016-12-19 20:50:36
tags:
---

 - sso examples on the net only for SAML / Google Suite
 - no sso examples for simple web identities like google
 - almost everyone has a google account
 - best practise to use roles instead of hardcoded users and access keys
 
 - cloudformation template
 - sam powered, or not. no inline code
 - inline code for keeping it simple
 
 
 
 
 
 references:
 
  - https://blog.jayway.com/2016/08/17/introduction-to-cloudformation-for-api-gateway/
  - http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-custom-url.html
  - http://stackoverflow.com/questions/39772259/how-do-i-cloudform-an-api-gateway-resource-with-a-lambda-proxy-integration