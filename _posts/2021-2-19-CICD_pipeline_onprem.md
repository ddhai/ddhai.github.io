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

CI/CD Branching strategy
![Pic2](/images/cicd-pipeline.jpeg)
Chart representing tagging of docker images and helm charts
![Pic3](/images/cicd-strategy.png)

- MASTER, develop, support branch is locked and no direct commits are allowed. Commits are only allowed through pull requests from the feature/bugfix branch and respectively from the develop to MASTER branch.

## Deployment Environments
- **PR (pull request) Environment**, which is a Kubernetes cluster used for deploying artifacts from feature/bugfix branch deployments for each microservice/repository.
- **STG(Staging) Environment** which is a Kubernetes cluster used for deploying artifacts from develop branch on merge from feature/bugfix branch, for each microservice/repository.
- **Platform-master** which is a Kubernetes cluster used for deploying artifacts from the master chart (umbrella chart helm) and deploy the entire application

## Bitbucket Configuration in Jenkins

![Pic4](/images/cicd-jenkins-1.png)

- Creating multibranch pipeline job
Jenkins > New item

### Jenkins Job configuration

![Pic5](/images/cicd-jenkins-2.png)
![Pic6](/images/cicd-jenkins-3.png)


- Click Save once done with configuration
Master-chart deployment job
Jenkins > New item

![Pic7](/images/cicd-jenkins-4.png)
![Pic8](/images/cicd-jenkins-5.png)
![Pic9](/images/cicd-jenkins-6.png)

## Bitbucket repository structure to support CI/CD
All the repositories will have a cicd directory inside the root folder and it will contain the below files/folders and scripts which perform different functions:
.
├── README.md
├── cicd
│ ├── helm
│ │ └── dummy-service
│ │ ├── Chart.yaml
│ │ ├── templates
│ │ │
│ │ └── values.yaml
│ ├── jenkins
│ │ ├── Jenkinsfile.groovy
│ │ ├── config.yaml
│ │ ├── stage_build.sh
│ │ ├── stage_docker.sh
│ │ ├── stage_release.sh
│ │ ├── stage_test.sh
│ │ └── version.sh
│ └── sonar-project.properties
├── docker
│ ├── Dockerfile
