

Problem:

 - reference resources created (e.g. in ebextensions) so that they can be provided the Elastic Beanstalk environment as an envrioment variable and to make it known to the application.


Solution:

Elastic Beanstalk environment variables are actually nothing more than an option setting for the environment.

 - define environment variables in cloudformation:
 
 ```
 ```
 
 For example, a Redis Cache cluster
 
 
 Another possibility would be to use Route53 Private Hosted Zones
