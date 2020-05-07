---
layout: post
title: A quick and dirty introduction to interacting with containerd
---

*This article will be updated as I have more time to explore some more features of ctr* :)

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

The output will look pretty familiar if you've used "kubectl get" or "docker ps".

`ctr --namespace k8s.io containers info 0b41398e452dd3eb18a2137b85bc270df63c3c826eeb3cace55ba40b43cc9e21 | jq`

You'll notice that that this is essentially the yaml file used to create the pod, just in a slightly different format but could easily be reconstructed. This most closely resembles the "-o json" in the kubectl command.

Have you noticed that we've provided a namespace flag to the ctr command? I was at first stumped when I thought that listing containers without a namespace would implicitly mean "--all-namespaces", however it does not work this way for ctr. There is more information on the topic and implementation [here](https://github.com/containerd/containerd/blob/master/docs/namespaces.md), but the benefits here are that it makes multi-tenancy easier*.

#### Inspecting running containers

This may be limited to KIND, but I found that when I was attempting to use kubectl after accessing the nodes, it didn't work. Specifically, I wanted to quickly check the logs of a container, where I had been messing with the static manifest files in /etc/kubernetes/manifests... so, I started exploring different ctr commands.

##### Exec-ing into a container is a little bit more difficult.

1. Get a list of all tasks from a container listed above

`ctr --namespace k8s.io tasks ls`

From this output we have a list of tasks, these are just "processes" and these make up our Pods. I tried to spawn a shell from each of those tasks, while some failed, a few succeeded.

2. Execute a task, which is essentially a Pod, the atomic unit of Kubernetes. Importantly, it wouldn't let me execute with only the task hash below, it needs the corresponding process ID.

`ctr --namespace k8s.io tasks exec --exec-id 16537 --tty e791befbe4f877cf96ee044c111e3428958120f519f17dfb3e2e0b696b6d6666 /bin/sh`

3. I tried to imagine if I did not have kubectl logs at my disposal so the following directory helped here.

`docker exec -ti <control plane hash> /bin/sh \
cd /var/log/pods/<pod-to-be-inspected>`

I've included some of the output below for brevity. 

```
ctr --namespace k8s.io tasks exec --exec-id 674 --tty 3a239f4388ff2e512f25d7a9a451350d603b8a2657506cdbabbde52f428d9595 /bin/sh
# ls
bin   dev  home  lib64	mnt  proc  run	 srv  tmp  var
boot  etc  lib	 media	opt  root  sbin  sys  usr
# cd usr/local/bin
# ls
clean-install  etcd-3.2.24  etcdctl-3.0.17  etcdctl-3.4.3	  noop
etcd	       etcd-3.3.17  etcdctl-3.1.12  invoke-rc.d		  runlevel
etcd-3.0.17    etcd-3.4.3   etcdctl-3.2.24  migrate		  update-rc.d
etcd-3.1.12    etcdctl	    etcdctl-3.3.17  migrate-if-needed.sh
```
   *Example output from spawning a shell into the etcd pod*

```
ctr --namespace k8s.io tasks ls
TASK                                                                PID     STATUS
cf790c378f350850e7b06ae9bb9bf01f234ec83d41c42e313e0d3049d3bd9d28    485     RUNNING
bc1e6dd080f812df2d30ca58ae7f4fe6804c4914d9f99507033b038ccc910d87    868     RUNNING
4ba4dc9c8ff5fa5932de50cdd1f52f6e1b03339f6f2238d299839638591ffe07    1260    RUNNING
8b8858179ad7025569a70733310764334de29a2a9101c1216aeb91a4e15066f9    1320    RUNNING
13d589105d4853e03149432390339522246b2777ba91562ab41649eedf9e9dc3    1393    RUNNING
3a239f4388ff2e512f25d7a9a451350d603b8a2657506cdbabbde52f428d9595    674     RUNNING
0ee8edce311455cfe9ef13d3681795d6067bdd0fbe53fcc95788e37cbc50d11e    917     RUNNING
96bbd28d856166d588d3cf9999a50fb579553a6a046965b1126cb1e122ad6438    1244    RUNNING
f3bb725c435c3354bc764e3d01e2d01852c8f8b180cae2ef96c23c9b6ab5eb6a    598     RUNNING
f904f71946f33feb9e5360f5c61cb3ce72dc6e79e493676093baa1edcbbcb89a    963     RUNNING
658d96b33be7140ddc2a7dde46f651da16a3c76808f70c8f121dd74a3ac78eb2    1332    RUNNING
2db1b0dcb2235a41b2545a3b611434c79165cf2725ae6093b6b49c54722936e9    3765    RUNNING
9242a2ebb305f7c58ed7becb7bda599b8c57ffb8feef27bd0e783d87e803fb6e    3799    RUNNING
76a93f6c941a0a8f5471c0624aef6244ab3a3a8638999f807dcbbce00f2e165a    378     RUNNING
7219ed5acddd9405bb9648a34605cfded68a27fb9e6d86cf3a25ef0a2c2a3c47    505     RUNNING
0b41398e452dd3eb18a2137b85bc270df63c3c826eeb3cace55ba40b43cc9e21    634     RUNNING
9fb6f1f477983b8495c9e02d8e13ad03f8d845a12bf325b42abbc4194d4fe958    860     RUNNING
a96d89027c98ed12daeb1ce326b3c45544cc24508a67617e293d7cc3482194ab    1254    RUNNING
```
   *Example of task listing output*
   


#### Interacting with containerd in Golang

Note: I found this a little difficult at first, as my first foray into using third party packages and getting away from the standard libraries (think "fmt" *cries internally*). If there are any suggestions to the below, feel free to reach out to me.

##### Running a container with a shell

Given that the containerd website has a fairly good example of how to interact with containerd through Golang, I used the example here to develop my understanding of how I might go about generating the "Container" metadata objects that are then associated with a "Task", tasks are the process used to start your containers, and the Container metadata object is just the configuration specification that task should execute when started. Before moving on credit goes to the containerd maintainers and developers for those resources, which are on their [website](https://containerd.io/docs/getting-started/), under "Getting Started"

The program I used is [here](https://github.com/VariableExp0rt/containerd-golang) and I'll paste the snippet of code that I changed to get a shell on an Nginx container. 

```
...snip...
        container, err := client.NewContainer(
                ctx,
                "nginx-test",
                containerd.WithImage(image),
                containerd.WithNewSnapshot("nginx-snapshot", image),
                containerd.WithNewSpec(oci.WithSpecFromFile("config.json")),
)
...snip...
```
What does this mean? Line by line, I've described that below;
.. 1. Initialise the container metadata object with the client method .NewContainer()
.. 2. Set the context, as in the example, we did previously for client.Pull()
.. 3-4. Set the Image to be used by the container, in mine I left it "image" but you could easily give it the name "nginx"
.. 5. A new snapshot is set for any storage that will be used by the container
.. 6. Set the oci.SpecOpts you might need (which are mapped to the oci-runtime specs in this package [here](https://godoc.org/github.com/opencontainers/runtime-spec/specs-go)).

Why have I used a WithSpecFromFile option? Well, because I was having some issues figuring out what options and use (and of course the occassional type error ;) because I'm still learning here) I realised that you could also use *runc* to generate a spec for you in json! For me, I needed a default spec so I passed the following command;

`runc spec --rootless` 

From wherever you ran that command, and perhaps it's easier from within your project directory, you'll have a new file called "config.json" (If you like json better you could probably just edit this file as needed and not need to mess with the oci package from containerd, since it's essentially a 1-1 mapping from my initial glance).

`cat config.json | jq` (now you have some of the nice highlighting (if that's you're thing ;)!)

Note: I actually changed the "terminal" option to "false", which seemed to be needed to pass through. I wondered why, and this post I found (credits to the author) seems to do well at explaining Standard IO in the context of runc. [more reading!](https://medium.com/@Mark.io/runc-terminal-and-run-modes-c677d7027335)

So by supplying the configuration file to containerd, when it talks to runc it uses the mappings to essentially generate this file on the fly, configures your container to this specification and starts it (provided you supply a task to it... and start it...). Runc is the container runtime used by containerd, more on that topic, which I'll looking into too, is [here](https://www.opencontainers.org/)

` go build main.go`
` sudo ./main` 

![A sHeLl]()

I found that without executing this as root, you'll have a hard time connecting to the containerd.sock socket (waaaaaah). By now you'll be staring at a no doubt blinking cursor, and if you type whatever your usual command is when dropping into a shell (ls?) then you'll see we're actually in the container (I know the functionality is a bit rough, so I'm still figuring out how to get a fully interactive tty).


That's all for now, I might do something with Open Policy Agent next, or how to *securely* configure the metadata objects with the oci.SpecOpts options :)

Peace

