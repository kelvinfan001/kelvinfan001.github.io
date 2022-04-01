---
title: Containerizing Your Dev Environment
date: 2022-03-31
tags: []
description: Tutorial on containerizing your development environment using the Remote-Containers VS Code extension.
tag:
  - Docker
  - containers
  - VS Code
  - tutorial
---

<img width="auto" src="/assets/img/dev-container/docker_vsc.png" alt="Docker + VS Code"/>

## Rationale

Having used [Fedora Silverblue](https://docs.fedoraproject.org/en-US/fedora-silverblue) for a year, one thing I missed as I return to the proprietary world of the Walled Garden of Apple (_for shame_) is the idea of an "immutable" host and containerizing everything.

> Unlike other operating systems, Silverblue is immutable. This means that every installation is identical to every other installation of the same version. The operating system that is on disk is exactly the same from one machine to the next, and it never changes as it is used.
Silverblue’s immutable design is intended to make it more stable, less prone to bugs, and easier to test and develop.

Silverblue encourages (forces) you to containerize everything, and that includes your development environment. I no longer work from a Linux machine, but I've decided to continue following this philosophy.

### Clean Host

This workflow allows you to keep your host in a pristine state. For example, aside from a handful of GUI apps, which are sandboxed anyway, my personal Mac has only Docker installed on it. This is important, as installing packages directly on your host can make it difficult to disentangle which binaries and libraries are from your OS provider and which are user-installed. On Linux systems, traditional package managers aren’t aware of the "base bootimage". This leads to many hard-to-find bugs and unexpected consequences during updates. Note: For Macs, starting from macOS Catalina, critical OS file locations like `/usr/bin/` are [mounted on a dedicated read-only system volume](https://support.apple.com/en-ca/HT210650) to combat this exact issue by default. However, this creates its own set of problems, as outlined in point 2 of [this comment](https://news.ycombinator.com/item?id=26037930) from a Homebrew maintainer.

### Freedom to be Fearless

Ever copy-pasted some command you barely understand from a blog or StackOverflow answer, and just crossed your fingers and held your breath as you pressed the `Enter` key in your terminal? If something breaks while you're developing inside a container, you can simply throw that container away and create a new one. Inside a development container, you can fearlessly install any package, and know that, in the worst-case scenario, you can "roll back" to its original state by creating a new container (provided that you kept an up-to-date development container Docker image).

### No more conflicts

You can easily create a dedicated environment for each project you have. Similar to the idea of [`venv`](https://docs.python.org/3/library/venv.html), except on the OS level. Your host no longer has to contain that random prerequisite package you had to install to follow a tutorial for one of your courses from two years ago.

### Unified Environment

Have everyone on your team be on the same OS (Linux distribution). You can further make sure that your development container's image installs the set of required development tools for your project. This should solve a lot of cases of "but it works on my machine". You can then check your Docker image for the project into source control or upload it to a container registry.

## How it works

On a high level, it's as simple as maintaining Docker images for each project or keeping a couple of long-running "pet" containers around. With bind-mounts, you can keep all your source code files on your host, and access/update them from the container. 

You can adjust the amount of integration between your host and your development container. For example, if you're on a Linux workstation, you can make working in your development container feel almost indistinguishable from working directly on your host by providing your container access to the host user's home directory, the Wayland and X11 sockets, networking, removable devices (like USB sticks), systemd journal, SSH agent, D-Bus, ulimits, /dev and the udev database, etc. (see [Toolbox](https://github.com/containers/toolbox)).

## Remote-Containers Extension

It's definitely possible to do all of the above manually, i.e. something like:

```
docker run -it -v $(pwd):/workspace development-container /bin/bash
```

This mounts the current working directory into a `/workspace` directory in the container and attaches to a Bash shell. However, this is cumbersome.

Luckily, if you use Visual Studio Code as your primary code editor, the [Remote-Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) makes it extremely easy to follow the containerized development environment workflow.

By automating a lot of what was described above, the extension integrates the container with the editor and terminal, allowing you to use VS Code as if everything were running on your host. For example, you can install your VS Code extensions into the container and use them as you would normally. It also allows you to customize exactly how much integration you want, which extensions you want pre-installed, etc., using the [`devcontainer.json`](https://code.visualstudio.com/docs/remote/devcontainerjson-reference) file. 

### Prerequisites

- Docker CLI and engine
- Visual Studio Code
- Remote-Containers extension

### Creating the `.devcontainer/` directory

You will need a `.devcontainer/` directory for each of your projects. This is the directory that Remote-Containers will look for and it is used to house the Dockerfile for the development container, as well as the `devcontainer.json` configuration file.

Get started by creating a directory to house your project, and create a sample file:

```
mkdir my-project && cd my-project
echo $'#!/bin/bash\necho "hello word"' > hello.sh
```

Now open up the command palette (⌘⇧P) and type in "Remote-Containers: Add Development Container Configuration Files".

![Add dev container config files](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cj5nf8f5optkuci5j9ej.png)

There are many base images you can get started from; let's assume we are creating a Rust project, and search for the Rust base image.

![Select base image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b85qdvs1qmoofg8v2ius.png)

For this image, it will then prompt you to select the OS version (the Rust Docker image happens to be Debian-based, so we will be selecting the Debian version here):

![Select OS version](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z6hxijebodxj6ygrtwjb.png)

The next step prompts you to quickly add features/extensions to your dev container. This is essentially a shortcut to editing the `devcontainer.json` config file, which we'll explore in more detail in a bit. Here I've selected the Fish shell and Git.

![Select features](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9160xd86g0zxg8jfwq4p.png)
 
The next few steps will prompt you to select the versions for these extra features.

After this, the `.devcontainer` directory will be generated, along with a Dockerfile and `devcontainer.json` file.

Let's take a closer look at `devcontainer.json`. It comes with a lot of useful comments explaining each of the fields.

```json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the README at:
// https://github.com/microsoft/vscode-dev-containers/tree/v0.231.1/containers/rust
{
  "name": "Rust",
  "build": {
    "dockerfile": "Dockerfile",
    "args": {
      // Use the VARIANT arg to pick a Debian OS version: buster, bullseye
      // Use bullseye when on local on arm64/Apple Silicon.
      "VARIANT": "buster"
    }
  },
  "runArgs": [
    "--cap-add=SYS_PTRACE",
    "--security-opt",
    "seccomp=unconfined"
  ],

  // Set *default* container specific settings.json values on container create.
  "settings": {
    "lldb.executable": "/usr/bin/lldb",
    // VS Code don't watch files under ./target
    "files.watcherExclude": {
      "**/target/**": true
    },
    "rust-analyzer.checkOnSave.command": "clippy"
  },

  // Add the IDs of extensions you want to be installed when the container is created.
  "extensions": [
    "vadimcn.vscode-lldb",
    "mutantdino.resourcemonitor",
    "matklad.rust-analyzer",
    "tamasfe.even-better-toml",
    "serayuzgur.crates"
  ],

  // Use 'forwardPorts' to make a list of ports inside the container available locally.
  // "forwardPorts": [],

  // Use 'postCreateCommand' to run commands after the container is created.
  // "postCreateCommand": "rustc --version",

  // Comment out to connect as root instead. More info: https://aka.ms/vscode-remote/containers/non-root.
  "remoteUser": "vscode",
  "features": {
    "git": "latest",
    "fish": "latest"
  }
}
```

[This section](https://code.visualstudio.com/docs/remote/devcontainerjson-reference#_image-or-dockerfile-specific-properties) lets you specify almost anything that you can with the Docker CLI. For example, the `runArgs` key lets you specify an array of Docker CLI arguments to be used when running the container.

The Dockerfile that it generates is also quite self-explanatory.

```Dockerfile
# See here for image contents: https://github.com/microsoft/vscode-dev-containers/tree/v0.231.1/containers/rust/.devcontainer/base.Dockerfile

# [Choice] Debian OS version (use bullseye on local arm64/Apple Silicon): buster, bullseye
ARG VARIANT="buster"
FROM mcr.microsoft.com/vscode/devcontainers/rust:0-${VARIANT}

# [Optional] Uncomment this section to install additional packages.
# RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
#     && apt-get -y install --no-install-recommends <your-package-list-here>
```

Let's edit the `devcontainer.json` file and Dockerfile to support "Docker from Docker".

As an aside, you might already be wondering, "what if I want to use Docker and Docker-Compose during development?". While it is possible to install the Docker engine inside your development container and run "Docker in Docker", this is not recommended for the number of reasons explained [here](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/). What makes more sense is to install _only_ the Docker CLI and Docker-Compose in your development container, and, from within the container, invoke your _host_'s Docker engine to launch new containers.

Let's modify the Dockerfile to install the Docker CLI by adding the following:

```Dockerfile
# Install Docker CLI.
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && apt-get -y install \
    ca-certificates curl gnupg lsb-release \
    # libssl-dev libudev-dev pkg-config zlib1g-dev llvm clang make build-essential \
    && curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian bullseye stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null \
    && apt-get update && apt-get -y install docker-ce-cli="5:20.10.12~3-0~debian-bullseye"
# Install Docker Compose.
RUN curl -sSL "https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/docker-compose
```

Next, we will need to bind mount the Docker socket in the container to the host's Docker socket to allow us to control the host's Docker engine from the dev container.

Normally, using the Docker CLI, we would do:

```
docker run -v /var/run/docker.sock:/var/run/docker.sock ...
```

However, we can also do achieve the same thing via `devcontainer.json`. Simply add the `mounts` key. We would also need to connect to the container as root for this to work.

```json
{
    "name": "Rust",
    "build": {
        "dockerfile": "Dockerfile",
        "args": {
            "VARIANT": "bullseye"
        }
    },
    "runArgs": [
        "--cap-add=SYS_PTRACE",
        "--security-opt",
        "seccomp=unconfined"
    ],
    "mounts": [ "source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind" ],

...

    "remoteUser": "root"
}
```

Now, open up the command palette again and type in "Rebuild and Reopen in Container".

![Rebuild and reopen in container](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x00naxbck6z8app8i2ut.png)

This will build the container from the Dockerfile, and then run it as specified by `devcontainer.json`.

Go to the VS Code terminal, and try checking the OS release file:

```bash
root ➜ /workspaces/my-project $ cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 11 (bullseye)"
NAME="Debian GNU/Linux"
VERSION_ID="11"
VERSION="11 (bullseye)"
VERSION_CODENAME=bullseye
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

The file that we created from the host is there, as well:

```bash
root ➜ /workspaces/my-project $ ls
hello.sh
```

We can also start Docker containers right from inside the development container:

```bash
root ➜ /workspaces/my-project $ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
7050e35b49f5: Pull complete 
Digest: sha256:bfea6278a0a267fad2634554f4f0c6f31981eea41c553fdf5a83e95a41d40c38
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

...
```

Doing a `docker ps`, you can see that the VS Code-created dev container is running with the name "vsc-\<name-of-your-project-root-dir\>". You'll also see the other containers running on your host (remember, we are connected to the host's Docker engine).
```bash
root ➜ /workspaces/my-project $ docker ps
CONTAINER ID   IMAGE                                             COMMAND                  CREATED         STATUS         PORTS     NAMES
c3ceb43b1f81   vsc-my-project-0a087c264c6455c09d6a06823954d1e2   "/bin/sh -c 'echo Co…"   2 minutes ago   Up 2 minutes             keen_knuth
```

VS Code also automatically forwards ports in your container to the host, so, for example, if you're hacking on a web app, you can start a server from your development container, and pointing your host's browser to the right port will just work.

## Conclusion

That was a quick look at why it's worth your time containerizing your development workflow; luckily, with the Remote-Containers extension, it really shouldn't take much time at all :)

There's also a bunch of other cool stuff that the Remote-Containers extension can do, like attaching to an existing container; from there, you can install all your favourite VS Code extensions. It can even work with other container engines like [Podman](https://docs.podman.io/en/latest/). I highly recommend checking it out at https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers!
