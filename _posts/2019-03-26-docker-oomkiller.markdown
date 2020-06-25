---
title:  "Behind the Docker Daemon has the OOM-Killer"
date:   2019-03-26 16:36:37 +0100
categories: Docker Kubernetes Kernel
tags:
  - devops
  - Docker
---

Recently I have written an [article](https://mrmuggymuggy.github.io/datadog/monitoring/jmx/kubernetes/datadog-jmx/) about Datadog on Kubernetes, The Datadog jmxfetch(java) process got killed by the host OOM-killer due to the JVM heapsize over allocation, I was surprise on the discovery. I knew that the containers and the host shared the same kernel space, I thought that docker daemon would just fail the container instead of actively intervene by killing the process.

Java has actually adapte to the extensive container usage nowadays by adding `XX:+UseCGroupMemoryLimitForHeap` option, it is really handy that the applications allocate the heapsize according to the container’s allocated resources, it allows a better application scalability. You can find plenty of articles with detailed explications.

What this article interest on, is how to find out the container resource limits and usage in a introspect way. Sometime, inside your application you just need to know the current resource limits/usage, Like how JVM knows the cgroup limits.

In the unix based containers you can find out the cgroup limit and usage in /sys/fs/cgroup

| ![memory limit and usage inside your docker container]({{ site.url }}{{ site.baseurl }}/assets/images/docker-oomkiller/cgroup1.png)
|:--:|
| *memory limit and usage inside your docker container* |

upon you can find the total memory limit, and in memory.stat you can find the total RSS memory usage at the moment for all the processes running inside the docker container.
cgroup contains also many other system metrics, such as cpu, devices and pids, the whole eco-system is a bit complex, a good documentation can be found on [link](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt).
one last thing, you can find the pid mapping from container to host with /pid/$pid/cgroup

| ![alt]({{ site.url }}{{ site.baseurl }}/assets/images/docker-oomkiller/cgroup2.png)
|:--:|
| *container to host pid mapping* |

it is interesting.