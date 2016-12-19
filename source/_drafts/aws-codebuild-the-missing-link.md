---
title: aws-codebuild-the-missing-link
date: 2016-12-19 20:50:36
tags:
---

Deployment pipelines are very common today, as they are usually part of a continiuous deployment workflow. While it's possible to use for example Jenkins, or ConCourseCi for those pipelines, I prefer using services in order to minimize operations and maintenance and to concentrate on generating business value. Luckily, AWS has a service called CodePipeline which makes it easy to create deployment pipelines with several steps ...
 
Using Lambda as Build Steps

Before CodeBuild was released, the only way to test build projects with pure serverless AWS components (so no VM or container management involved) was to use Lambda functions as a custom build step. That approach had several drawbacks, though:

- 5 minutes maximum execution
- using NPM very awkawrd