---
layout: post
title: "How I do releases with Elixir and Docker"
date: 2017-09-22
---

![complicated](/images/rubenvent1.jpg)



> I am going to assume you have knowledge of [Elixir](https://elixir-lang.org/), [Docker](https://www.docker.com/) and [Distillery](https://hexdocs.pm/distillery/getting-started.html) for this article.


When combining Elixir releases and Docker, size is the name of the game.  For example, Elixir's default Docker image is around 888MB.  My sample application for this article is around 20MB.  Using almost a gig for 20Mb seems wasteful, no?  Deploying or pulling a large Docker image takes time.  I can see I have convinced you that this is a problem.

Now, I know you are probably saying, "Use [Alpine Linux](https://alpinelinux.org/)" instead Brad.  I would respond with, "you are correct occasional reader".  For those who don't know, Alpine Linux is a tiny Linux distribution.  The official Alpine Docker image is around 4 MB in size.  Pretty good huh? 

## Begin

There are a couple of things I would like to cover in this article.

* First, use Docker to build our release in the same OS environment that will be used in production (Alpine Linux)

* Second, use the newly created release in a minimal Docker image without all the Elixir/Erlang run time


## The process

We are going to build our Elixir release on a Alpine Linux Docker image (plumbing included).  Then we will copy that release off and into a vanilla Alpine Docker image for use in production.


## The setup

> I will once again assume you have an Elixir application with Distillery all setup and good to go.  


We will need to create 2 docker files for our two step approach--this will make more sense later on.  One file for building the release and the other file for building the final image.

I like to call the build Dockerfile...wait for it...`Dockerfile.build`. 


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

Here we are using [bitwalker's Alpine Linux Elixir image](https://github.com/bitwalker/alpine-elixir).  This image contains Erlang and Elixir's run time making it possible to build our release.  There are a few packages missing for our application, so I will use Alpine's package manager, `apk`, to install them.

The next thing you might notice is the `ARG` keyword.  This allows us to pass the `mix version` of our application to the build process.   Please feel free to insert your own method to get the version of said application into the build.

I have used and seen: mix task's or git tags.


#### Dockerfile

The next file is our final image for our release.

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

In this `Dockerfile` we are using the stock Docker Alpine Linux image--which is what bitwalker's is based off.  I create a directory to untar our release from the build portion.  After that, I setup the configuration file and environment variables.
The release runs in the foreground.  This will be handy if Docker is using syslog driver for logs.

## The Glue

Let's take a second to recap.  We have 2 docker files in which we will use to create a `production ready` Docker image of our application.

How do we use these two files to do this?  Glad you asked.  We will use some bash glue to pull all this together.

>  There are a ton of ways one could do this, but for brevity I chose to use a bash script.  

This script will build our release via our `Dockerfile.build` and then copy the release to it's new home via the `Dockerfile`.


#### build_realease.sh

```bash
#!/bin/bash

BUILD_TAG=$1
VERSION=$2
TEMP_NAME=`uuidgen -t | cut -d'-' -f1`
RELEASE_NAME=$3
FINAL_TAG=$4

if [ "$#" -ne 4 ]; then
	echo "Illegal number of parameters"
	echo "------Usage-----"
	echo "./build_release.sh build-tag version release-name final-tag"
	exit 2
fi

# build release
echo -e "\e[33mBuilding release...\e[0m"
docker build --no-cache -t $BUILD_TAG -f Dockerfile.build --build-arg version=$VERSION .
# create container from image
echo -e "\e[33mCreating temporary container...\e[0m"
docker create --name $TEMP_NAME $BUILD_TAG
# copy release off
echo -e "\e[33mcopying $RELEASE_NAME from build image...\e[0m"
docker cp $TEMP_NAME:/tmp/$RELEASE_NAME ./$RELEASE_NAME
# remove temp container
echo -e "\e[33mRemoving temp container...\e[0m"
docker rm $TEMP_NAME 
# build final image
echo -e "\e[33mBuilding final images...\e[0m"
docker build --no-cache -t $FINAL_TAG .
```

As you can see, the script expects 4 command line arguments passed in.

1. Docker image tag--For example, `bradleydsmith/sample-application-build`.
2. Application version.
3. Release name--For example, `sample_application.tar.gz` 
4. Final Docker tag--For example, `bradleydsmith/sample-application`

I use standard Docker commands to achieve the image build, container life cycle, and artifact copying.  Please see [Docker documentation](https://docs.docker.com/) for further help.

Showtime.  This is what is looks like when executing it from the command line.  I added some colors for the shell output since building a release is very chatty.

```bash
./build_release.sh bradleydsmith/sample-application-release 0.1.0 sample_application.tar.gz bradleydsmith/sample-application
# omitted for space
==> Release successfully built!
    You can run it in one of the following ways:
      Interactive: _build/prod/rel/sample_application/bin/sample_application console
      Foreground: _build/prod/rel/sample_application/bin/sample_application foreground
      Daemon: _build/prod/rel/sample_application/bin/sample_application start
 ---> c0c7a9312fee
Removing intermediate container 2643b5bd5b5b
Step 9/9 : CMD /bin/sh
 ---> Running in ac47b9629a89
 ---> 8e67488f6872
Removing intermediate container ac47b9629a89
Successfully built 8e67488f6872
Successfully tagged backstopit/sample_application-release:latest
Creating temporary container...
b66c7597a18ea63ba69ff720a560f72ead4bc5220bf9e11e33bde55af9fa5f31
copying sample_application.tar.gz from build image...
Removing temp container...
6707d622
Building final image...
# omitted for space
```

If things go well, you should have the release in the application directory and a Docker image with the tag you provided.  Let's check our handy work.

```bash
$ docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
backstopit/check_uptime            latest              03209618eb51        10 hours ago        50.9MB
backstopit/check_uptime-release    latest              429e9b1c6927        10 hours ago        430MB
elixir                             latest              99275bfdda58        3 days ago          888MB
...

```

Our production ready Docker image is only 50.9MB!  Deploy time should be really fast now.  If you look at our build image, it is a whopping 430MB.


## Going forward

There is more work to do here for a stable build tool.  One thing you could do is convert all this logic into a `mix task`.
Another approach would be to wire this into your CI/CD pipeline. Alter the script--or replace it--with actions to push the release to a object store like S3.  Then in the production `Dockerfile` use the `ADD` keyword to pull down the release from the object store.  

Until next time...
