# Dockerfile Best Practices 

Writing production-ready Dockerfiles is not as simple as you could think about it. 

This repository contains some best-practices for writing Dockerfiles. Most of them are coming from [my experience](https://twitter.com/DamianNaprawa) while working with Docker & Kubernetes. 
This is all guidance, not a mandate - there may sometimes be reasons to not do what is described here, but if you _don't know_ then this is probably what you should be doing.

## Best practices included in the Dockerfile

The following are included in the Dockerfile in this repository:

- [Use official Docker images whenever possible](#use-official-docker-images-whenever-possible)
- [Alpine is not always the best choice](#alpine-is-not-always-the-best-choice)
- [Limit image layers amount](#limit-image-layers-amount)
- [Run as a non-root user](#run-as-a-non-root-user)
- [Do not use a UID below 10,000](#do-not-use-a-uid-below-10-000)
- [Use a static UID and GID](#use-a-static-uid-and-gid)
- [The `latest` is an evil, choose specific image tag](#the-latest-is-an-evil-choose-specific-image-tag)
- [Only store arguments in `CMD`](#only-store-arguments-in-cmd)
- [Always use COPY instead of ADD (there is only one exception)](#always-use-copy-instead-of-add-there-is-only-one-exception)
- [Always combine RUN `apt-get update` with `apt-get install` in the same run statement](#always-combine-run-apt-get-update-with-apt-get-install-in-the-same-run-statement)

## Use official Docker images whenever possible

Using official Docker images dedicated for your technology should be the first choice. Why? Because those images are optimized and tested by MILLIONS of users. 
Creating your image from scratch is a good idea when official base image contains vulnerabilities or you cannot find a base image dedicated for your technology.

Instead of installing SDK manually:

```Dockerfile
FROM ubuntu
RUN wget -qO- https://packages.microsoft.com/keys/microsoft.asc \
| gpg --dearmor > microsoft.asc.gpg \
&& sudo mv microsoft.asc.gpg /etc/apt/trusted.gpg.d/ \
&& wget -q https://packages.microsoft.com/config/ubuntu/18.04/prod.list \
&& sudo mv prod.list /etc/apt/sources.list.d/microsoft-prod.list \
&& sudo chown root:root /etc/apt/trusted.gpg.d/microsoft.asc.gpg \
&& sudo chown root:root /etc/apt/sources.list.d/microsoft-prod.list
RUN sudo apt-get install dotnet-sdk-3.1
```

Use official Docker Image for dotnet

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster
```

## Alpine is not always the best choice

Even though Alpine is lightweight, there are some known issues with performance for some technologies (https://pythonspeed.com/articles/alpine-docker-python/).

Second thing about  Alpine-based images - security. Most of vulnerabilities scanners do not find any vulnerabilities in Alpine-based images. If scanner didn't find any vulnerability, does it mean it's 100% secure? Of course not. 

Before making a decision, evaluate what are benefits of alpine. 


## Limit image layers amount

Each `RUN` instruction in your Dockerfile will end up creating an additional layer in your final image. The best practice is to limit the amount of layers to keep the image lightweight.

Instead of:

```Dockerfile
RUN curl -SL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.gz" --output nodejs.tar.gz
RUN echo "$NODE_DOWNLOAD_SHA nodejs.tar.gz" | sha256sum -c -
RUN tar -xzf "nodejs.tar.gz" -C /usr/local --strip-components=1
RUN rm nodejs.tar.gz
RUN ln -s /usr/local/bin/node /usr/local/bin/nodejs
```

Use this:

```Dockerfile
RUN curl -SL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.gz" --output nodejs.tar.gz \
&& echo "$NODE_DOWNLOAD_SHA nodejs.tar.gz" | sha256sum -c - \
&& tar -xzf "nodejs.tar.gz" -C /usr/local --strip-components=1 \
&& rm nodejs.tar.gz \
&& ln -s /usr/local/bin/node /usr/local/bin/nodejs
```

## Run as a non-root user

Running containers as a non-root user substantially decreases the risk that container -> host privilege escalation could occur. 
This is an added security benefit ([Docker docs](https://docs.docker.com/engine/security/#linux-kernel-capabilities)).

For Debian-based images, removing root from container can be done like this:

```Dockerfile
RUN groupadd -g 10001 dotnet && \
   useradd -u 10000 -g dotnet dotnet \
   && chown -R dotnet:dotnet /app

USER dotnet:dotnet
```

**NOTE**: Sometimes when you remove the root from container, you will need to adjust your application/service permissions.

For example, dotnet applications cannot run on port 80 without root privileges and you have to change the default port (in the example it's 5000).

```Dockerfile
ENV ASPNETCORE_URLS http://*:5000
```

## Do not use a UID below 10,000

UIDs below 10,000 are a security risk on several systems, because if someone does manage to escalate privileges outside the Docker container their Docker container UID may overlap with a more privileged system user's UID granting them additional permissions. For best security, always run your processes as a UID above 10,000.

## Use a static UID and GID

Eventually someone dealing with your container will need to manipulate file permissions for files owned by your container. If your container does not have a static UID/GID, then one must extract this information from the running container before they can assign correct file permissions on the host machine. It is best that you use a single static UID/GID for all of your containers that never changes. We suggest `10000:10001` such that `chown 10000:10001 files/` always works for containers following these best practices.

## The `latest` is an evil, choose specific image tag

Use a specific image `version` using `major.minor`, not `major.minor.patch` so as to ensure you are always:

1. Keeping your builds working (`latest` means your build can arbitrarily break in the future, whereas `major.minor` _should_ mean this doesn't happen)
2. Getting the latest security updates included in new images you build.

### Why you perhaps shouldn't pin with a SHA

SHA pinning gives you completely reliable and reproducible builds, but it also likely means you won't have any obvious way to pull in important security fixes from the base images you use. If you use `major.minor` tags, you get security fixes by accident when you build new versions of your image - at the cost of builds being less reproducible.

**Consider using [docker-lock](https://github.com/safe-waters/docker-lock)**: this tool keeps track of exactly which Docker image SHA you are using for builds, while having the actual image you use still be a `major.minor` version. This allows you to reproduce your builds as if you'd used SHA pinning, while getting important security updates when they are released as if you'd used `major.minor` versions.

## Only store arguments in `CMD`

By having your `ENTRYPOINT` be your command name:

```Dockerfile
ENTRYPOINT ["/sbin/tini", "--", "myapp"]
```

And `CMD` be only arguments for your command:

```Dockerfile
CMD ["--foo", "5", "--bar=10"]
```

It allows people to ergonomically pass arguments to your binary without having to guess its name, e.g. they can write:

```sh
docker run IMAGE --some-argument
```

If `CMD` includes the binary name, then they must guess what your binary name is in order to pass arguments etc.

## Always use COPY instead of ADD (there is only one exception)

Arbitrary URLs specified for ADD could result in MITM attacks, or sources of malicious data. In addition, ADD implicitly unpacks local archives which may not be expected and result in path traversal and Zip Slip vulnerabilities. 

Even if ADD can lower the number of image layers, COPY should be used whenever possible.


## ALWAYS COMBINE RUN APT-GET UPDATE WITH APT-GET INSTALL IN THE SAME RUN STATEMENT
Using `apt-get update` alone in a RUN causes caching issues and subsequent `apt-get install` instructions fail. It’s related to caching mechanism that Docker use. While building the image, Docker sees the initial and modified instructions as identical and reuses the cache from previous steps. As a result, the apt-get update is not executed because the build uses the cached version. Because the `apt-get update` is not run, the build can potentially get an outdated version of packages.

```sh
RUN apt-get update && apt-get install -y \
    curl \
    git \
    build-essential  \
    && rm -rf /var/lib/apt/lists/*
```


