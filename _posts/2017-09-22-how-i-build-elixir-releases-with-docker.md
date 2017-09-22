---
layout: post
title: "How I do releases with Elixir and Docker"
date: 2017-09-22
---

![complicated](/images/rubenvent1.jpg)


* I am going to assume you have decent knowledge of [Elixir](https://elixir-lang.org/), [Docker](https://www.docker.com/) and [Distillery](https://hexdocs.pm/distillery/getting-started.html).

When building Elixir releases in Docker, size is the name of the game!  For example, Elixir's default image is around 888MB.  Hardly efficient to run in production.  "Use [Alpine Linux](https://alpinelinux.org/)", you scream.  

That is exactly what we are going to do.  There are a couple of things I would like to cover in this article.

* First, use Docker to build our release in the same OS environment that will be used in production

* Second, use the newly created release in a Docker image without all the Elixir/Erlang runtime

## The process

We are going to build our Elixir release on a Alpine Linux Docker image.  Copy that release off and into a new, shiny, smaller Docker image for use in production.


## The setup

* I will once again assume you have an Elixr application with Distillery all setup and good to go.  

We are going to start by creating 2 separate Dockerfiles.  One for building the release and the other for building the Docker image for production.

I like to call the build Dockerfile...wait for it...`Dockerfile.build`.  Here are the contents which we will go over.

#### Dockerfile.build

```bash
FROM bitwalker/alpine-elixir


RUN apk --update add git build-base curl gawk openssl-dev
ADD . /tmp

ENV RELX_REPLACE_OS_VARS true
ENV REPLACE_OS_VARS true
ENV MIX_ENV prod
ARG version


RUN cd /tmp && \
    mix local.hex --force && \
    mix local.rebar --force && \
    mix deps.get && \
    mix compile && \
    mix release --env=prod && \
    cp _build/prod/rel/sample_application/releases/$version/sample_application.tar.gz /tmp/
```

Here we are using bitwalker's Alpine Linux Elixir image.  This image has almost everything we need to build our release.  Our sample application needs a few extra things.  I use `apk` to install them.

There is nothing out of the ordinary in this Dockerfile except the `version` ARG.  Please feel free to use and other means to get the version of said application.  Other uses might include a mix task or git tags.


#### Dockerfile

```bash
FROM alpine

RUN mkdir -p /opt/sample_application && \
    apk --update add bash libcrypto1.0 && rm -rf /var/cache/apk/*
ADD sample_application.tar.gz /opt/sample_application
ADD config/system.config /opt/sample_application/config

ENV RELX_REPLACE_OS_VARS true
ENV REPLACE_OS_VARS true
ENV MIX_ENV prod
ENV CHECK_UPTIME_HOME /opt/sample_application

CMD ["/opt/sample_application/bin/sample_application", "foreground"]
```

In this Dockerfile we are using the stock Docker Alpine Linux image--which is what bitwalker's is based off.  I create a directory to untar our release.  After that, I setup the config file and environment variables.  The release runs in the forground for logging purposes.

## The Glue

Let's take a second to recap.  We have 2 dockerfiles in which we will use to create production ready Docker image of our application that can be stored on your favorite image store.

We need some glue to put all this together.  There are a ton of ways one could do this, but for brevity I chose to use a bash script.  

This script will build our release via our `Dockerfile.build` and then copy release to it's new home via the `Dockerfile`.


#### build_realease.sh

```bash
#!/bin/bash

TAG=$1
VERSION=$2
TEMP_NAME=foo
RELEASE_NAME=$3

if [ "$#" -ne 3 ]; then
	echo "Illegal number of parameters"
	echo "------Usage-----"
	echo "./build_release.sh tag version release-name"
	exit 2
fi

# build release
docker build --no-cache -t $TAG -f Dockerfile.build --build-arg version=0.1.0 .
# create container from image
docker create --name $TEMP_NAME $TAG
# copy release off
docker cp $TEMP_NAME:/tmp/$RELEASE_NAME ./$RELEASE_NAME
# remove temp container
docker rm $TEMP_NAME 
```

As you can see, the script assumes some command line arguments will be passed in.  The first is the tag.  For example, `bradleydsmith/sample-application-build`.

The second is the `version` of our application.  Remember it is used in our `Dockerfile.build`.

Third, is the release name.  For example, `sample_application.tar.gz` 

I then use standard Docker commands to acheive the build.  Please see [Docker documentation](https://docs.docker.com/) for further help.

This is what is looks like all together

```bash
./build_release.sh bradleydsmith/sample-application-release 0.1.0 sample_application.tar.gz
```


OK, we successfully build our release and it is sitting in a tagged 





