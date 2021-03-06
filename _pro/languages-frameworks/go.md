---
title: Go on Docker
weight: 48
tags:
  - go
  - languages
  - docker
category: Languages &amp; Frameworks
redirect_from:
  - /docker-integration/go/
---

* include a table of contents
{:toc}

In this article you will learn about setting up a Go-based project on Codeship Pro.

## Services and Steps
Before reading through the documentation please take a look at the [Services]({% link _pro/getting-started/services.md %}) and [Steps]({% link _pro/getting-started/steps.md %}) documentation page so you have a good understanding how services and steps on Codeship work.

## Dockerfile
We will start by creating a Dockerfile that lets you run your Go based test suite in Codeship.

Please take a look at our [Dockerfile Caching best practices]({{ site.baseurl }}{% link _pro/getting-started/caching.md %}) article first to make sure you build your Dockerfile in a way that only invalidates the Docker cache when necessary.

Following is an example Dockerfile with inline comments describing each step in the file.

```Dockerfile
# Starting from the latest Golang container
FROM golang:latest

# INSTALL any further tools you need here so they are cached in the docker build

# Set the WORKDIR to the project path in your GOPATH, e.g. /go/src/github.com/go-martini/martini/
WORKDIR /go/src/your/package/name

# Copy the content of your repository into the container
COPY . ./

# Install dependencies through go get, unless you vendored them in your repository before
# Vendoring can be done through the godeps tool or Go vendoring available with
# Go versions after 1.5.1
RUN go get
```

This Dockerfile will give you a good starting point to install any further packages or tools you might need. Take a look at our [browser testing documentation]({{ site.baseurl }}{% link _pro/continuous-integration/browser-testing.md %}) to find and install any further tools you might need for your build.

## codeship-services.yml

The following example will use the Dockerfile we created to set up a container we call project_name (please change to your specific project name) that will run your build. We're adding a [PostgreSQL container](https://hub.docker.com/_/postgres/) and [Redis container](https://hub.docker.com/_/redis/) so the tests have access to those two services.

When accessing other containers please be aware that those services do not run on localhost, but on a different hostname, e.g. "postgres" or "mysql". If you reference localhost in any of your configuration files you have to change that to point to the hostname of the service you want to access. Setting them through environment variables and using those inside of your configuration files is the cleanest approach to setting up your build environment.

```yaml
project_name:
  build:
    image: organisation_name/project_name
    dockerfile: Dockerfile
  # Linking Redis and Postgres into the container
  links:
    - redis
    - postgres
  # Set environment variables to connect to the service you need for your build. Those environment variables can overwrite settings from your configuration files if configured. Make sure that your environment variables and configuration files work together as expected.
  environment:
    - DATABASE_URL=postgres://postgres@postgres/YOUR_DATABASE_NAME
    - REDIS_URL=redis://redis
# Service definition that specify a version of the service through container tags
redis:
  image: redis:2.8
postgres:
  image: postgres:9.4
```

For more information about other services you can use with Codeship check out our [services and databases documentation]({{ site.baseurl }}{% link _pro/getting-started/services-and-databases.md %}).

## codeship-steps.yml

Now we're going to set up our steps file.

Every step in a build gets its own clean container and linked services. Any setup commands that are necessary to setup a linked container (e.g. database migrations) need to be run for every step. While this duplicates some of the work, it also makes sure your parallelized test commands run completely independently.

As a step can only have a single command the best way to set up the build is through scripts in your repository. You can then call those scripts in your step file.

We're simply going to call `go build ./...` and `go test ./...` with `bash -c ""`. You can also use a Makefile, or simply create your own script in your repository that gets called as part of the build.

```yaml
- service: project_name
  command: bash -c "go build ./... && go test ./..."
```

## Parallelizing your build

If your build contains different commands that can be run in parallel independently you can make your build faster by running those steps in parallel. An example would be the following:

```yaml
- name: ci
  type: parallel
  steps:
  - service: project_name
    command: bash -c "go build ./... && go test ./..."
  - service: project_name
    command: OTHER_INDEPENDENT_COMMAND_OF_YOUR_BUILD
```
