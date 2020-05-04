---
layout: post
title: A quick and dirty introduction to interacting with *containerd*
---

### Environment; KIND clusters

This blog post is to reinforce some of the things I learnt about [ctr](https://github.com/containerd/containerd), the command-line interface for containerd, a container runtime interface that is the intermediary component between Docker and Runc. In this post, I'm using an incredibly simple setup on my local machine, in the form of a KIND cluster. KIND (Kubernetes in Docker) is a great way to begin learning about the different components of k8s, although I found it to be a little limited in some aspects, due to the contraints of *ctr* which I'll describe below.

Example config.yaml for a minimal cluster, straight from the [KIND project](https://kind.sigs.k8s.io/docs/user/quick-start/#creating-a-cluster) website:

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```
There are additional settings you can supply, including disabling the default CNI, exposing the master (if you absolutely have to) and using custom pod cidr ranges.

#### Containerd basics

Well, what _can_ we do with ctr? It offers some basic functionality to identify running containers;
`ctr --namespace k8s.io containers ls`

#### Inspecting running containers

This may be limited to KIND, but I found that when I was attempting to use kubectl after accessing the nodes, it didn't work. Specifically, I wanted to quickly check the logs as a container, where I had been messing the static manifest files in /etc/kubernetes/manifests... so, I started exploring different commands with ctr;



![_config.yml]({{ site.baseurl }}/images/config.png)

