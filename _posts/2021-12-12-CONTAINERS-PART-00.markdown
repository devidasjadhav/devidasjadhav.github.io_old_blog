---
layout: post
title:  "Container internals Deep Dive 00"
date:  2021-12-12 12:00:03 +0530 
categories: bcc
---

Past year I have been looking under the hood of Docker. I came across multiple interesting things. This series is my attempt to share whatever I have learned during past year.
I expect this series that will start new discussions. than privding answers.

## Before we begin.

This series is intended for theose who are interested in understanding "How Docker works ?".
This who want to just run docker.
getting started can be easy with

    docker pull <image name>  # to pull from docker registry i.e. docker hub
    docker run <image name > # to run local image. if not it will pull from registry.

```bash
$ sudo docker run -it ubuntu bash
[sudo] password for dev: 
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
7b1a6ab2e44d: Pull complete 
Digest: sha256:626ffe58f6e7566e00254b638eb7e0f3b11d4da9675088f4781a50ae288f3322
Status: Downloaded newer image for ubuntu:latest
root@3fdeef5a4a1e:/# 
```
Thats it. If you want to know behind the scenes. You are welcome to this series.
Sit tight and follow this series.


## History of Docker.

Why are we learning history of containers ?

Lets get streight to architecture of containers.

Believe me history is best way to get started of understanding how we reach here. 

Importenetly it will answer the question why we do what we do ? or why are containers designed this way ?


# What are containers by the way ?

Glad you asked. There are many defination. this is from [Docker website](https://www.docker.com/resources/what-container)

> A container is a standard unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another. A Docker container image is a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries and settings.


Important statement here is " container image is a lightweight, standalone, executable package of software "

Container is simply process running with
1.  its own just enought root file system
2.  running with bare minimal previlages
3.  contained in sandboxed environment so that it will not harm Host.


# Chroot.

It all started with chroot system call.
`chroot` system call was added to unix by Bil Joy in 1982. almost 3 decades ago. With intention to test installation and build system.

as per man page ( `man 2 chroot`)

> chroot - run command or interactive shell with special root directory

In nutshell chroot will allow you to switch your root file system. (i.e. what you see in `ls /`)

# How chroot relates to containers ?

Note: Hyperlinks in this section I have pointed out to source code. Which you may like to take a look at.

Lets investigate. with source code.

[Moby Project](https://github.com/moby/moby) is an open-source project created by Docker to enable and accelerate software containerization.

Moby / Docker uses containerd to manage containers through [libcontainerd](https://github.com/moby/moby/tree/master/libcontainerd).

Libcontainerd in init uses [go-runc](https://github.com/containerd/containerd/blob/master/pkg/process/init.go#L38)

go-runc is wrapper for [runc](https://github.com/opencontainers/runc)

runc [init](https://github.com/opencontainers/runc/blob/master/init.go#L13) leads to [chroot](https://github.com/opencontainers/runc/blob/master/libcontainer/rootfs_linux.go#L141)

Summary: 

1. Moby / Docker is cli. which is frontend talking to dockerd on unix socket (/var/run/docker.sock) .
2. containerd ( one of many container runtime ) communicates with docekerd on gRPC API exposed by [containerd](https://github.com/containerd/containerd/blob/main/services/server/server.go#L104)
3. containerd being container runtime does many tasks pulling image , storing image , etc. once of them is running container. For which it uses cli utility by opencontainers called runc. 
4. runc is responsible for spawning and running containers. which is done through wraper go-runc.
5. runc when spawning new container uses chroot. as pointed out above.


# So What ?

I could have just said docker uses chroot. but I was really interested in flow how new container is spawned hence all information above. just to aprecite and understand this technology.

Containers use chroot for special purpose.

# Why change root in first place ?

Changing root is neccessary to solve one of main problem which docker intended to solve. 
Solving "It works for me!" issue. Having something reproducible.

So solve that issue. we use new RFS which containes all dependancy files needed.
With chroot we can run it without conflicting with host libraries and binaries.

Hence runing process inside will use all resources in container Root FS. rather than Host Root FS.

## And much more.

This was about chroot. which we will explore more in next post. 
purpose of this post is to share plan for 10 next posts in this series.

1. Chroot (already discussed above)
2. Cgroups ( limit resource uses )
3. Namespace ( Isolating containers running on same system )
4. Network Namespaces and CNI ( Special Namespaces which is critical part of Container ecosystem )
5. containerd ( details of containerd and internals )
6. OCI standard ( Why and What of OCI )
7. runc and crun ( implementations of OCI and comaparing them )
8. rootless containers ( case study of rootless containers using podman )
9. Kata containers 
10. firecracker-microvm 

These are only 10 planned sure there are plenty of tools and architectures to write about.
I am commiting to these 10 for now. See you guys in next post about Chroot.

