---
title: "CI/CD Part 3: Building containers with Docker"
subtitle: ""
summary: ""
authors: []
tags: [CI/CD, Gitlab, YAML, Docker]
categories: []
date: 2023-05-30T02:14:14-07:00
lastmod: 2023-05-30T02:14:14-07:00
featured: false
draft: false
---

This is the third post in a mini-series on [designing Gitlab CI/CD pipelines]( {{< relref "/post/cicd-1-config/index.md" >}} ).  In the last [post]( {{< relref "/post/cicd-2-packages/index.md" >}} ), we discussed setting up your `.pypirc` and `.netrc` files in the context of a Gitlab CI/CD pipeline to enable building and pushing packages to a package registry, as well as for installing code from a private registry.  In this post, we'll discuss putting together a Dockerfile for building containers in the context of a Gitlab CI/CD pipeline.

I'm using what's called a "multi-stage" build.  This is exactly what it sounds like -- it breaks the process of building a Docker image into multiple stages.  In doing so, we often have the benefit of a final image that is smaller than a single-stage build, because we only include the artifacts needed to run our containerized application.  

Similarly, we can leverage multi-stage Docker builds to minimize duplicated code in Dockerfiles.  For example, let's say we have a scenario where we want to build an image for a `Production` environment as well as a `Test` environment.  The `Test` environment might include some additional dependencies, scripts, exports, etc. that the Production environment doesn't.  Instead of creating two Dockerfiles, one for each environment, we can define a single-stage that encompasses the overlapping parts of both the `Production` and `Test` images, and then define the extra stuff in a separate stage to build the `Test` image.

In the example below, we have a three-stage Docker build, with stage names:
 * `base`: sets up some basic environment variables
 * `python-deps`: installs your package and creates a virtual environment
 * `runtime`: the actual application you want to run, with only the necessary files for running it

#### Create a base image

```yaml
# set base image
# bigger base images yield slower image load times, and have more security vulnerabilities
FROM python:3.9-slim as base

# install virtual environment in ${project_dir}/.venv
ENV PIPENV_VENV_IN_PROJECT 1
# dont write .pyc files
ENV PYTHONDONTWRITEBYTECODE 1
# get some more information about faults when building images
ENV PYTHONFAULTHANDLER 1
```

#### Install package dependencies

If you were building your Docker image locally, you'd have access to any authentication tokens or SSH keys necessary to pull from remote or private repositories.  However, Docker is naive to these variables -- we have to explicitly provide them at build time. Within the Dockerfile, we define three environment variables using the `ARG` keyword:
 1. `CI_DEPLOY_USER`
 2. `CI_DEPLOY_PASSWORD`
 3. `CI_JOB_TOKEN`

If you think that these variables look familiar, you're right.  They're the same pre-defined variables that exist in the context of a Gitlab CI/CD pipeline that act as authentication tokens for a `.pypric` file and `.netrc` -- they're utilized by the `setup_tokens.sh` script to setup the `.pypirc` and `.netrc` files *within* the Docker image.

```yaml
# image for installing dependencies
# we only need .venv and app `runtime` image, not all other bloat
FROM base AS python-deps

####################################
# ----------------------------------
ARG CI_DEPLOY_USER
ARG CI_DEPLOY_PASSWORD
ARG CI_JOB_TOKEN
# ----------------------------------
####################################
```

These variable are available outside of the Docker image, but not within the image itself, so we need to "show" them to Docker at build time via the `--build-arg` flag:

```bash
docker build --build-arg CI_DEPLOY_USER=$CI_DEPLOY_URDER \
             --build-arg CI_DEPLOY_PASSWORD=$CI_DEPLOY_PASSWORD \
             --build-arg CI_JOB_TOKEN=$CI_JOB_TOKEN \
             ...
             ...
```

Now they are contained within the Docker image and can be provided to the `setup_tokens.sh` script, which then allows us to pull packages down from our remote package registry.  We also no longer need the SSH keys, since we're authenticating through Gitlab itself.

```yaml
# install pipenv in `python-deps` image
RUN python3 -m pip install pipenv
RUN apt-get update \
    && apt-get install --yes --no-install-recommends gcc g++ libffi-dev

# Dependency installation looks a little different for local packages
WORKDIR /home/app


# copy files to `python-deps` image
COPY setup.py setup_tokens.sh Pipfile Pipfile.lock ./
# copy over application-specific code that you want to install
# this is unique to my specific project -- use your own directories here
COPY templateci/ templateci/

# run setup_tokens script to setup .pypirc and .netrc within image
RUN chmod +x ./setup_tokens.sh && ./setup_tokens.sh

# authentication tokens are now available to pipenv
RUN python3 -m pipenv install --deploy --dev

# get rid of unnecessary libraries after install
RUN apt-get autoremove --yes gcc g++ libffi-dev \
    && rm -rf /var/lib/apt/lists/*
```

#### Create your final runtime image

Above, we created the virtual environment that allows our application to run.  As such, we no longer need the raw source code or any other random files that were contained in the original project directory that might have been needed to build the virtual environment.  Now we create a stage called `runtime` in which we copy over the generated virtual environment from the previous stage

```yaml
# image for running the application
FROM base AS runtime

# Copy virtual environment from `python-deps` image to `runtime` image
COPY --from=python-deps /home/app/.venv /.venv
# add virtual environment to PATH
ENV PATH="/.venv/bin:$PATH"

# Create new user -- app will run as new user
RUN useradd --create-home -u 1099 user
WORKDIR /home/user/app
USER user

COPY . .

CMD ["python3", "-m", "pytest"]
```

This is the actual image that get's run when we call

```bash
docker run ${IMAGE_NAME}
```