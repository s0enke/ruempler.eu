
Dieser Blog Post soll zeigen, wie 

Problem:

In a Multi account setup, pipelines may cross the boundaries of severeal AWS accounts. Imagine for example a Continous Infrastructure Pipeline, which first deploys to a staging env, and if everything worked out, it continues to deploy to production. 

AWS has documentation for Cross Account, diese ist aber nicht sehr intuitiv, und beschraenkt sich auf CodeDeploy.