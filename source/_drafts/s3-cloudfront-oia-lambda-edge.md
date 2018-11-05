

 - default root object works only for the root, not for subfolders.
 - Workaround is to enable static website hosting
  - which cannot be used with OIA 
 - das ding von last week in aws, war wieder son komisc hes geloet
## Why Terraform?

 - Terraform adds the burden of maintaining the state, CloudFormation usually preferred
 - Terraform more feature complete
 - Multi Region AWS
 - check cloudonaut article!


## Learnings

 - CloudFront seem to cache Lambda@Edge errors, e.g. when there is an error in the Lambda function

## References

 - https://serverfault.com/questions/581268/amazon-cloudfront-with-s3-access-denied
 - https://forums.aws.amazon.com/thread.jspa?threadID=84964
 - https://serverfault.com/questions/581268/amazon-cloudfront-with-s3-access-denied
 - https://read.acloud.guru/supercharging-a-static-site-with-lambda-edge-da5a1314238b
 - https://aws.amazon.com/blogs/aws/new-amazon-cloudfront-feature-default-root-object/
