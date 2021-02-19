---
 layout: post
 tags: K8s, jenkins, sonar, cicd
 title: Continuous Integration and Continuous Deployment (CI/CD) on Kubernetes with Jenkins (On-premise)
---

Continuous integration/continuous delivery/continuous deployment helps the engineering team ship software in a faster and safer manner. CI/CD differs across organizations depending on their use-case.

In this blog post, we will cover a step by step procedure we followed at Fortna to set up a CI/CD Pipeline using Jenkins and deploy on Kubernetes.
Sections in this post:
- Tooling for the CI/CD pipeline
- Branching strategy employed
- Deployment Environments
- Bitbucket Configuration in Jenkins
- Jenkins Job configuration
- Bitbucket repository structure to support CI/CD
- Branching scenarios and expected outcome from CI/CD
- Conclusion

Tools used in this pipeline

![Pic1](/images/cicd-architecture.png)

- Bitbucket on-premise
- Jenkins server
- Sonarqube on-premise
- Harbor registry
- Kubernetes on-premise

We will be focusing on setting up the pipeline.
## Branching strategy employed
The branching strategy that designed for our development purpose:

![Pic2](/images/cicd-pipeline.jpg)
