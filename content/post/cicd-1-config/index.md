---
title: "CI/CD Part 1: Gitlab Pipelines"
subtitle: "Setting up config files"
summary: ""
authors: []
tags: [Python, CI/CD, Gitlab, YAML, Docker]
categories: []
date: 2023-05-02T02:14:14-07:00
lastmod: 2023-05-02T02:14:14-07:00
featured: false
draft: false
---

I recently developed a template workflow to help our team adopt a CI/CD-based development strategy.  Many of our web applications and tools were based on simple repository structures.  With growing datasets and ever-increasing use by outside teams, we found ourselves needing to add new features more frequently to many of these tools and believed that continuous integration and deployment could help us not just develop more quickly, but also more intelligently.  Since we use Gitlab to store our code, we decided to use the Gitlab CI/CD tools.  

Documentation on much of this process was scattered and/or sparse, so I decided to put what I learned and implemented into a more coherent set of notes.  The current post relates to a basic setup of the `.gitlab-ci.yml` file and using Gitlab CI/CD.  The next few posts are about what I learned in the process of setting this up, where I'll discuss:

 * [Setting]( {{< relref "/post/cicd-2-packages/index.md" >}} ) up authentication for pushing packages to a package registry and for pulling down private repositories
 * [Designing]( {{< relref "/post/cicd-3-docker/index.md" >}} ) Dockerfiles for building images using CI/CD
 * [Pushing]( {{< relref "/post/cicd-4-cr/index.md" >}} ) images to a container registry

## Setup

I did all of my testing using my personal Gitlab account.  To separate things out, I created a new Project called "Package Registry”, as well as a test repository that was used for building a local Python project called "TemplateCI".  

The "Package Registry" Project serves as just that -- an all-inclusive location for any software packages your CI/CD pipelines build.  You can find the built packages by clicking **${Project Name} > Deploy > Package Registry**.  Every "Project" in Gitlab has the ability to store packages in its own registry, but I felt it cleaner to store everything in one repo.  Similarly, the actual code that I'll be packaging will be stored in the "TemplateCI" repo.

## Basic jobs

The basis of a Gitlab CI/CD pipeline is the `.gitlab-ci.yml` file, which is composed of a set of explicitly-defined and temporally-ordered “stages”.  A stage is composed of a set of “jobs”.  Jobs are the workhorses of the CI/CD pipeline, and define explicit tasks that a CI/CD pipeline runs.  By default, all jobs from one stage run in parallel, unless specified otherwise (using the `needs` keyword as an attribute of a job induces a temporal directed acyclic graph – jobs can be made “dependent” on the successful completion of other jobs within a stage).  In this example, I’ve defined three stages:

 1. run-unit-tests
 2. build-package
 3. build-image

Any job associated with the `run-unit-tests` stage will run to completion (or failure) PRIOR TO ANY job in the stages `build-package` and `build-image` starting.  If all jobs in the `run-unit-tests` stage complete successfully, then the next stage (`build-package`) will begin.  We define individual jobs, and give them a stage attribute.  Stages define the rough ordering of jobs.  Each job runs in the context of an “image” or environment.  We can set a global `image` and define stages as

```yaml
# global CI/CD image
image: python:3.9-slim

# stages of this example pipeline
stages:
  - run-unit-tests
  - build-package
  - build-image

 variables:
    LC_ALL: C.UTF-8
    LANG: C.UTF-8
```

or define the image as an attribute of a job.  Gitlab CI/CD by default uses Docker images in which to run jobs.  Setting the image is analogous to using the `FROM` command in a Dockerfile.  We've also set some global variables, here the `LC_ALL` and `LANG` variables.

To “run” Gitlab pipelines for the purpose of CI/CD, we use “runners”, which are build instances installed on a server.  Gitlab offers “shared” runners (use of these is free if you use www.gitlab.com, but you need to register a credit card to prevent abuse of Gitlab resources).  You can also register your own device(s) to act as a Gitlab runner.

Below are examples of two jobs in the `run-unit-tests` stage.  These two jobs are effectively the same code, apart from the unique unit tests that they run.  However, we've made the job `unit-tests-2` dependent on the output of the job `unit-tests-1` (see the `needs` keyword of `unit-tests-2`) .  Both jobs use the global `python:3.9-slim` image.  We can run some “setup”  stuff (`before_script`), run an actual script (`script`), and run clean up (`after_script`, not shown) -- these delineations (before, during, after) are for organizational purposes, and not due to any explicit functional differences in the delineations.

```yaml
### Ignore the script called "setup_tokens.sh" for now -- we'll discuss this in another post. ###


# example job #1
unit-tests-1:
  stage: run-unit-tests
  before_script:
    - chmod +x ./setup_tokens.sh; ./setup_tokens.sh
    - python3 -m pip install pipenv
    - apt-get update
    - apt-get install --yes --no-install-recommends gcc g++ libffi-dev
    - python3 -m pipenv install --deploy --dev
  # run first set of unit tests
  script:
    - python3 -m pipenv run pytest -k 'test_examples1.py'
 

# example job #2
unit-tests-2:
  stage: run-unit-tests
  before_script:
    - chmod +x ./setup_tokens.sh; ./setup_tokens.sh
    - python3 -m pip install pipenv
    - apt-get update
    - apt-get install --yes --no-install-recommends gcc g++ libffi-dev
    - python3 -m pipenv install --deploy --dev
  # run second set of unit tests
  script:
    - python3 -m pipenv run pytest -k 'test_examples2.py'
  # wait for job 1 to finish
  needs: [unit-tests-1]
```

## Conditional pipeline jobs

The above jobs are relatively simple and will run every time you push a repository to Gitlab.  However, sometimes, we might only want to run a job if certain conditions are met.  For example, we might only want to build a package from the `main` branch, or only after a merge request is made.  To this end, we can add “rules” to a job that restrict when it is actually run.

Below is a more complicated job example.  The overarching goal of this job is to build a Docker image from a local Python project and push the image to the Gitlab Container Registry.  There’s a lot going on here, so I’ll break it up into pieces, but here is the whole job:

```yaml
# Conditional job
# Building a docker image and pushing this container to a container registry
# Link to main image: https://github.com/bentolor/docker-dind-awscli

# Conditions:
# --- merge request events
# --- target branch of merge request is "main"

# job name
build-image-glcr:
  
  # stage of pipeline
  stage: build-image
  # image that job is based on
  image: docker:20.10.16
  # sub-services of job
  services:
    - docker:20.10.16-dind
  # variables available to the job
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
    
  # some setup scripts -- here, just making sure docker is available
  before_script:
    - docker info

  # meat of the job -- authentication, building, run, push
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build 
      --build-arg CI_DEPLOY_USER=$CI_DEPLOY_USER 
      --build-arg CI_DEPLOY_PASSWORD=$CI_DEPLOY_PASSWORD 
      --build-arg CI_JOB_TOKEN=$CI_JOB_TOKEN
      -t $IMAGE_TAG .
    - docker run $IMAGE_TAG
    - docker push $IMAGE_TAG

  # job conditions
  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event' && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main"
```

This job is associated with a new stage called `build-image` that will run as the last stage of this example CI/CD pipeline.  Without going into the specifics of the [Dockerfile]( {{< relref "/post/cicd-3-docker/index.md" >}} ) just yet, this stage builds and pushes a Docker image to a remote repository.  We defined the job image as `docker:20.10.16` and an additional “service” attribute as `docker:20.10.16-dind` where “dind” means “Docker-in-Docker”.  The Docker-in-Docker feature allows an image to run Docker itself (pretty meta, huh).  The idea here is to instantiate a job from a specific Docker container (“image”) which itself has Docker installed (“dind”), which will allow the docker:20.10.16 image to build *another* Docker container.  We’ve also defined some variables:

 * `DOCKER_TLS_CERTDIR`: needed to allow the larger scale image to communicate with a service (honestly, I don’t quite understand this and documentation on CI/CD "services" is sparse)
 * `IMAGE_TAG`: refers to the container destination and the image “name” -- this just makes our lives easier by turning into a variable what would otherwise be a really long string

```yaml
#### Build Docker image and push to Gitlab Container Registry
build-image-glcr:
  stage: build-image
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
```

Next up, we have all the script stuff.  You’ll probably recognize most of the `docker ${command}` commands.  We first “authenticate” the current job with the Gitlab container registry (apparently there are a variety of ways to “authenticate” with Docker using one set of variables or another – the way below is the way I was able to get working, but there are [other solutions](https://stackoverflow.com/questions/61251622/how-to-authenticate-to-gitlabs-container-registry-before-building-a-docker-imag)).  We then build the Docker image, run the image, and push the image to the container registry.

```yaml
script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  - docker build 
    --build-arg CI_DEPLOY_USER=$CI_DEPLOY_USER 
    --build-arg CI_DEPLOY_PASSWORD=$CI_DEPLOY_PASSWORD 
    --build-arg CI_JOB_TOKEN=$CI_JOB_TOKEN
    -t $IMAGE_TAG .
  - docker run $IMAGE_TAG
  - docker push $IMAGE_TAG
```

You’ll notice a bunch of variables that I didn't explicitly define anywhere:
 * `CI_REGISTRY_USER`: username for project (I think we can also use `CI_DEPLOY_USER`, though it looks like there might be some things to figure out with branch / variable protections / non-protections)
 * `CI_REGISTRY_PASSWORD`: defaults to `CI_JOB_TOKEN` value (this value is ephemeral e.g. valid only for one job at a time I think?  I think we can also use the `CI_DEPLOY_PASSWORD` for a longer-lived alternative, though it looks like there might be some things to figure out with branch / variable protections / non-protections)
 * `CI_REGISTRY`: defaults to `https://gitlab.com/${group}/${project-name}/container_registry`
 * `CI_DEPLOY_USER`: generated user token
 * `CI_DEPLOY_PASSWORD`: generated token password
 * `CI_JOB_TOKEN`: see documentation [here](https://docs.gitlab.com/ee/ci/jobs/ci_job_token.html)


These are all [“predefined” variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html), meaning they already exist in the Gitlab CI/CD context as part of having a www.gitlab.com account, without you explicitly defining them.  However, I did run into some issues using `CI_DEPLOY_USER` and `CI_DEPLOY_PASSWORD`.  In addition to having predefined variables provided by Gitlab CI/CD, we can also [*manually* predefine variables](./docs/SettingEnvVariables.md) for a whole Gitlab Project or for a whole Group.  Go the page for your **Project/Group > Settings > CI/CD > Variables > Expand**.  For example, I defined the `CI_DEPLOY_USER` and `CI_DEPLOY_PASSWORD` variables for my Group.  These variables are the Group-level authentication tokens and are now made accessible to all Gitlab CI/CD jobs running under this Group.  Additionally, although not shown here (because it's used within the `setup_tokens.sh` script), we've also defined an environment variable called `PACKAGE_REGISTRY_ID` that tells the CI/CD pipeline where to build and push packages.

And finally, what we’ve been waiting for, conditional pipeline jobs:

```yaml
rules:
  - if: $CI_PIPELINE_SOURCE == 'merge_request_event' && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main"
```

As part of this job, I've defined a rule that indicates that this job should **only** be run 1) upon a merge request event (i.e. if you click “Create Merge Request” in the console), and 2) if the name of the target branch of that merge request event is `main`.  For example, if I create a new branch called `dev` off of `main` and create a merge request in the Gitlab console, this job will run.  However, it’s important to note that, if you push a commit to the `dev` branch *after* creating the merge request, this job will *still* run, even if you havn’t created a *new* merge request e.g. `CI_PIPELINE_SOURCE` == "merge_request_event" will always default to `TRUE` after the first merge request event, as long as the source branch is still actively being developed.  Additionally, the job itself for this **will run the source branch code, not the target branch code**.  For Gitlab Premium users, there is an additional criteria called a “merged_result_event”, which would run the target branch (`main`) code after merging the source branch (`dev`) into the target branch.


Some of the (many) external links I used in this process:
 * [Predefined](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) CI/CD variables
 * [Adding](https://gitlab.com/gitlab-org/gitlab/-/issues/214014) variables to the Gitlab CI/CD context
 * [Setting](https://gitlab.com/gitlab-org/gitlab/-/issues/350582) up a ~/.netrc file with Gitlab credentials
 * [Executing](https://stackoverflow.com/questions/72789599/gitlab-ci-cd-execute-script-file-that-exist-in-the-repository) a bash scripting within a Gitlab CI/CD pipeline
 * [Passing](https://stackoverflow.com/questions/58939500/how-to-pass-gitlab-ci-file-variable-to-dockerfile-and-docker-container) variables to Docker image at build time
 * [Building](https://www.shellhacks.com/gitlab-ci-cd-build-docker-image-push-to-registry/) Docker image and pushing to registry
 * [Authenticating](https://docs.gitlab.com/ee/user/packages/container_registry/authenticate_with_container_registry.html) container registries
 * [Setting](https://docs.gitlab.com/ee/user/packages/pypi_repository/#authenticate-with-the-package-registry) up a ~/.pypirc files with Gitlab credentials