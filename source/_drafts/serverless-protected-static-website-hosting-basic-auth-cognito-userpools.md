 - One click template like Stelligent or cloudonaut

## Why not API Gateway

 - API gateway does not work with pathes - even if there are blog posts out that 
 

## Further thoughts

Lambda@Edge for the redirect if unauthenticated (oder gleich Basic Auth machen?)

## Learnings

Lambda@Edge does not work with origin error, e.g. 4xx or 5xx 

## tags:

serverless, htaccess, htpassword, password protection


## References

 - http://johnpreston.ews-network.net/posts/s3-and-cloudfront-basic-auth-hack/
 - https://gist.github.com/mjohnsullivan/31064b04707923f82484c54981e4749e
 - https://github.com/yegor256/s3auth
 - https://serverfault.com/questions/581268/amazon-cloudfront-with-s3-access-denied
 - https://blog.h4.nz/2017/01/20/a-cloudformation-custom-resource-for-cloudfront-origin-access-identities-oai/
 - http://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html
 
## Who tried to implement it as well?
 
- https://cloudncode.blog/2017/08/08/tidbit-api-gateway-as-a-s3-proxy-cloudformation-script-with-serverless-framework/ - security through obscurity, public read and writable 
- https://medium.com/@lmakarov/serverless-password-protecting-a-static-website-in-an-aws-s3-bucket-bfaaa01b8666