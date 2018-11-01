---
title: "Creating Your Own Istio (Part 1)"
date: 2018-09-19T14:30:07+01:00
lastmod: 2018-09-19T14:30:07+01:00
draft: false
keywords: []
description: "How we can add/remove features to existing micro-services just by adding/removing containers."
tags: [openshift, container, services, kubernetes ]
categories: [openshift, container, services, kubernetes ]
toc: true
image: https://github.com/cesarvr/hugo-blog/blob/master/static/static/logo/ocp.png?raw=true
---

Let say we have a micro-service exposing some business API and we want to gather some data about its usage such as how many calls, payload size, errors, response time, etc. Adding this feature would usually involves writing some code, some testing and re-deployment of a new version. But when you have multiple micro-services, this solution can be difficult to reuse.      

<!--more-->

To share this functionality with other services we can share the code in the form of a module or library, but unfortunately this will lock us down to a programming language, without mentioning that adding that module to a messy, untested, legacy code can involve to much unnecessary suffering. 

## Service Mesh 

Are you familiar with the ["Decorator Pattern"](https://en.wikipedia.org/wiki/Decorator_pattern)? This pattern is about adding behaviour at runtime to an individual object **without affecting the behavior** of other objects from the same class. That's my favorite part because implies that I don't need to see/touch the code; following then the "open for extension, close for modification" principle. 

We are are going to learn how to do this with running containers (services) in *Kubernetes/OpenShift*, by encapsulating the behaviour inside a container, and applying it to the services we want to decorate.     


## "Real World" Examples 

- [Istio](https://istio.io/).
- [Linkerd](https://linkerd.io/)


# Before We Start

This guide will be divide in three parts:

- **Part One**: We learn how to setup multiples containers in a single pod.   
- **Part Two**: We are going to write a simple server to read the usage pattern of any micro-service.  
- **Part Three**: Write a simple dashboard. Once this container is injected to other services, we are going to send some usage data to the dashboard with the usage information across our "service mesh".   

I'm going to use OpenShift because is the Kubernetes distro I'm most familiar with, but this techniques should also work in Kubernetes as well.

If you want to follow this guide you can install [oc-client](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md) with oc-cluster-up or even better make a free account in [OpenShift.io](https://manage.openshift.com). If you have trouble understanding some of the concepts, you read this [OpenShift getting started guide](https://github.com/cesarvr/Openshift).

When you deploy a service in Kubernetes/OpenShift you are making use of an entity called a pod, this entity provide a space for you application to run. The first thing we need to understand is that we can deploy multiple containers in the same pod. We are going to make use of this fact in the next post, for now here is a quick introduction.  

# Understanding The Pod

Pods are the building blocks to create applications in OpenShift, but for our purposes we can think of them as an container of containers, they provide a [isolation layer](http://cesarvr.github.io/post/2018-05-22-create-containers/) similar to Linux container. This means that containers running inside the pods believe they are running in a single machine.   

Like processes running in a "single machine", contained processes running inside the pod can communicate between each other using System V semaphore, POSIX shared memory or Linux sockets through "localhost".   


## Pod Creation 

This is a quick example of what a [pod](https://gist.github.com/cesarvr/3e80053aca02c7ccd014cbdfc2288444) looks like:

```xml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-pod
spec:
  containers:
  - name: my-container
    image: busybox
    command: ['sh', '-c', 'echo Hello World! && sleep 3600']
```

This YAML template defines a pod named *my-pod*, inside we are going to deploy a container using Busybox (a very small Linux distribution) base image. Then we are going display "Hello World" and sleep to keep the entire container alive for 3 thousand seconds.

We save this in a file called pod.yml and we execute the following command:

```sh
oc create -f pod.yml

# or you can create using a template stored somewhere else
oc create -f https://gist.githubusercontent.com/cesarvr/3e80053aca02c7ccd014cbdfc2288444/raw/52cde49116a6d6261a1f813034b957058180a7ee/pod.yml
```

That template creates the big jail (the pod) and once created it execute inside something similar to this: ```docker run -it busybox echo Hello World!; sleep 3600```.    

We can login into the pods running container using this *magic words*:

```sh
oc rsh my-pod
```

## Adding More Containers 

Adding a new container to existing pod is not difficult, we just need add a new entry in the template:

```xml
apiversion: v1
kind: pod
metadata:
  name: my-pod
  labels:
    app: my-pod
spec:
  containers:
  - name: first-container
    image: busybox
    command: ['sh', '-c', 'echo Hello World && sleep 3600']
  - name: second-container
    image: busybox
    command: ['sh', '-c', 'echo Hello World 2 && sleep 3600']
```

As you can observe that containers section is a collection, you can add as many containers as you want. After editing our template we just need to load the new template into OpenShift.

```sh
# if you create a pod before, you need to deleted first.
oc delete pod my-pod

# create it again.
oc create -f pod.yml

#or

oc create -f https://gist.githubusercontent.com/cesarvr/97a0139ca2dba9412254d9919da64e69/raw/5e593a9a4b9fff9af06c53670f939fd9caef94ff/pod.yml
```

Logging into the pod gets a bit trickier now as we need to specify the container, let say we want to login into the ```first-container``` container:

```sh
oc rsh -c first-container my-pod
```

If you want to login into the ```second-container```:

```sh
oc rsh -c second-container my-pod
```

## Communication Between Pods

### Simple Server

By now we should understand all the theory behind how the pod works, so let's put some of it into practice and deploy a simple web server using python, first we need to build our image:

```sh
oc new-build python~https://github.com/cesarvr/demos-webgl --name=web
```
Here we are using [new-build](https://cesarvr.io/post/buildconfig/) to create an image. We are going to provide this [repository](https://github.com/cesarvr/demos-webgl), where we got some simple HTTP pages.

We can check the location of our new image using the following command:

```sh
oc get is

# NAME          DOCKER REPO                                             TAGS
# web           docker-registry.default.svc:5000/web-apps/web           latest    
```

Now we need to update our template to add our newly created image:   

```xml
apiversion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-pod
spec:
  containers:
  - name: web
    image: docker-registry.default.svc:5000/web-apps/web
    command: ['sh', '-c', 'echo Hello World && sleep 3600']
  - name: proxy
    image: busybox
    command: ['sh', '-c', 'echo Hello World 2 && sleep 3600']
```

Here I just renamed containers name for clarity, we got the **web** container using our Python image we created before, and the second container to **proxy**. The only thing left is to edit the command section in the **web** container so we deploy a web server instead of just showing a message.

```xml
apiversion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-pod
spec:
  containers:
  - name: web
    image: docker-registry.default.svc:5000/web-apps/web
    command: ['sh', '-c', 'cd static && python -m http.server 8087']
 - name: proxy
    image: busybox
    command: ['sh', '-c', 'echo Hello World 2 && sleep 3600']
```

This command is very simple, once the container is created it will jump to the [static folder](https://github.com/cesarvr/demos-webgl/tree/master/static) and then run the [HTTP.Server module](https://docs.python.org/3/library/http.server.html).

The only thing remaining is to re-create our pods:

```sh
oc delete my-pod

oc create my-pod.yml
# or ...

oc create -f https://gist.githubusercontent.com/cesarvr/cecaf693a17b6f09b9eb3f5d38f33165/raw/2227781e4c3e71ecb68b22d052bdf8cd2c083c55/my-pod.yml
```

Now let's test that our containers can talk to each other inside the pod, for that we are going to the command [oc exec](https://docs.openshift.com/enterprise/3.0/dev_guide/executing_remote_commands.html) (which is similar to [docker exec](https://dzone.com/articles/docker-for-beginners) allow us to execute remote shell commands.

The syntax goes as follows:

```sh
oc exec -c <container-name> <pod-name> -- <shell-command>
```

Let's run [wget](https://www.computerhope.com/unix/wget.htm) Linux command to fetch the webpage in our **web** container:  

```sh
oc exec -c web my-pod -- wget -qO- 0.0.0.0:8087

# <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">  
```

This means that our web server is running, now let's test same remote command with the **proxy** container and we should get the same result:

```sh
oc exec -c proxy my-pod -- wget -qO- 0.0.0.0:8087

#<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd"
```

Here is the whole process:

![sidecar-deployment](https://raw.githubusercontent.com/cesarvr/hugo-blog/master/static/prometheus/sidecar-deployment.gif)

## Container Patterns

We should be able to create applications with multiple containers, also another important point is that we demonstrate that containers running inside the pod operate similar as it they where in a single machine, this is very important as we are going to use this in the next article to create our “Telemetry” container.  

If you want to know more about the container patterns you can take a look a this [paper](https://ai.google/research/pubs/pub45406). 