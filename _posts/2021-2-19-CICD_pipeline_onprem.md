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


## What does this script do!
- Jenkinsfile.groovy
This is linking to the Jenkins library that runs all the stages for the job the groovy file named **pipelineMicroserviceGeneric** have all the codes managing the pipeline

```
@Library('fortna-pipeline') _
pipelineMicroserviceGeneric()
```

fortna-pipeline — this alias name depends on the name that you provided for your jenkins library repository in jenkins configuration
- Jenkins > configuration > Global Pipeline Libraries
pipelineMicroserviceGeneric() — file in Jenkinslibrary which controls the pipeline workflow
- **config.yaml**
The configuration file defines a list of services to deploy, and for each specifying the target environments it should be deployed in, and optionally extra arguments for the tiller helm upgrade command.

![Pic7](/images/cicd-config.png)

- **name**: Name of your service (The names of the services must be the same as the helm chart names/directories)
- **environments**: To which environment (PR/STG) you want to deploy your service
- **targets**: To which cluster you need to deploy, trainer or engine, Depends on the target this will update the MASTER HELM CHART while merging to the master branch
- **extra_args**: pass extra arguments like config map on deployment, overwriting existing config maps.
- **notifications**: notification part in the config file is optional if you don’t want to get email notification apart from committer, skip the entire part from config.yaml, by default the committer will get an email (if you skipped this group)
- **Note**: Please configure the SMTP setting in Jenkins server for working of email notification (Jenkins > Manage Jenkins > Configure system > Extended E-mail Notification)

**stage_build.sh**
This script is used to build a package. If this stage is not required, the script. should not be present.
Example:
![Pic8](/images/cicd-bash-1.png)

stage_test.sh
The script that should run unit tests. This script is mandatory since all repositories should run unit tests. The script should exit with non-zero if any test fails.
Example:
![Pic9](/images/cicd-bash-2.png)

**stage_docker.sh**

- The script that should build one or more docker images.The docker image names when possible should be the same as the service name
Script called with the following arguments:
- `--full_build true|false` Jenkins DOCKER_FULL_BUILD build a parameter, to enable/disable caching of docker layers.
- `--prefix IMAGE_NAME_PREFIX` Prefix for the docker image, essentially indicating where the image should be pushed. The value would be one of: - xyz.internal.example.com/example/ - abc.internal.example.com/example/ - qwe.internal.example.com/example/ (Note: We are using 3 harbor registries and this depends on the branch)
- `--suffix TAG_SUFFIX` Suffix for the docker image tag, e.g. `-develop`, `-feature-my-branch`

![Pic10](/images/cicd-bash-3.png)

the values for `${PREFIX} & ${SUFFIX}` will be picked from the Jenkinsgroovy file which depends on the branch, `${PREFIX}` hold the values of harbor registry and `${SUFFIX}` hold the value of version number & tag

**stage_release.sh**

The script that performs any tasks required after the release e.g. push a new version of API to swagger, push python wheel package to PyPI, etc.
If this is not needed keep the script empty.

**sonar-project.properties**

The `sonar-scanner` the tool is called from the root of the repository, so all settings in the config file should be in relation to the root.
This will be in cicd directory (`/cicd/sonar-project.properties`)

![Pic11](/images/cicd-sonar.png)


**version.sh**

The script that used to get the version of the repository that used for tagging of docker images and helm charts as well as for various checks in the pipeline.
This script is mandatory.

![Pic12](/images/cicd-ver.png)


## Branching scenarios and expected outcome from CI/CD
While raising PR from a feature or bugfix branch to develop
### PR from feature/ or bugfix/ to develop branch

![Pic13](/images/cicd-br-1.png)

- Docker images and helm charts will get pushed to our PR harbor registry
- The deployment will happen to PR Kubernetes cluster

### On merge to develop

If the previous job is not successful then the merge is blocked on bit bucket using merge checked plugin

![Pic14](/images/cicd-br-2.png)

- Images/Helm charts pushed to staging harbor registry
- Deployment to STG environment

On merge to develop the pipeline will perform the Quality test and post coverage result in sonarqube-stg (http://example-harbor.example.com/). The build will fail depends on the Quality Gate.

![Pic15](/images/cicd-br-3.png)

**If the version is not found in the git tags the stage “Create released” will be executed**

In this stage, **Jenkins will build and push images and helm charts to QA harbor registry** with the tag as the version number.
It’s also possible to execute custom actions(for example to push packages to PyPI server) that can be added in **stage_release.sh**

If executed the step “Create released” will be highlighted

![Pic16](/images/cicd-br-4.png)

**On PR from develop to the master branch**

Checks that for the last commit a git tag exists named with the <version>, other stages will be skipped.
 
![Pic17](/images/cicd-br-5.png)

If the PR is raised after creating a release then this will be successful. Otherwise, the PR will fail as follows

![Pic18](/images/cicd-br-6.png)


**On merge to Master branch**


![Pic19](/images/cicd-br-7.png)

On successful merge to master branch only the “master chart update” will be executed and all the other steps will be skipped.
The step “master chart update” will update the **master-charts** with the new **microservice version** and the new version of the **master-charts** (umbrella chart)created(git tag + push in helm **QA** repo)

**Deployment of the platform from master-chart with updated service:**

On successful update of **master-charts**, a new Jenkins job will get triggered (<**Jenkins URL**> and deploy the entire application to platform-master cluster with the updated versions of microservices and you can instantly check.
 
### Support branch
Support branches can be used to make microservice releases that are not part of the normal release flow.
For example, they can be used for:
- Adding bugfixes or features to an old microservice release and release it bumping the patch version (example: from **1.12.14 to 1.12.15**) 
- Adding bugfixes or features to an old microservice release for a specific customer rollout (example: from **1.12.14 to 1.12.14-PRO001**) 
- If you are using bump version, In order to make bump2version work correctly with custom versions such as **1.2.3-ABC-2** which do not follow the normal major.minor.patch pattern it is required to modify the config. Look at **https://github.com/c4urself/bump2version/issues/147**
- Create a new microservice release starting or not starting from an already existing git tag
The majority of times a new support branch will be created starting from an existing git tag and will be called **support/\<meaningful-name\>** (example: **support/1.12.15, support/1.12.x, support/PRO001**)
The support branch works similarly to the develop branch, to make a change to the support branch we can use pull requests from **feature/** and **bugfix/ branches**
PR from feature or bugfix branch to support branch

![Pic20](/images/cicd-br-8.png)

- Images/Helm charts pushed to **pr harbor repo**
- No automated deployments

### On merge to support

![Pic21](/images/cicd-br-9.png)

- Images/Helm charts pushed to STG harbor
- No automated deployments

**If the version is not found in the git tags the stage “Create released” will be executed**

In this stage, Jenkins will **build and push images and helm charts to qa-repo** with the version as a tag.
It’s also possible to execute custom actions(for example to push packages to **PyPI** server) that can be added in **stage_release.sh**
If executed the step “Create released” will be highlighted

![Pic22](/images/cicd-br-10.png)

Source Code: 


## Conclusion
We have demonstrated a CI/CD workflow with Jenkins, Bitbucket on-premise, Sonarqube, Harbor, Helm and Kubernetes on-premise clusters. The benefit of this stack is flexibility since it allows you to implement practically any type of workflow quite conveniently.

