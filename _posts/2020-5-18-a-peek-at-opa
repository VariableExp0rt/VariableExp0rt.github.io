---
layout: post
title: A peek at Open Policy Agent
---

#### Introduction to Open Policy Agent

Open Policy Agent (OPA) is a "general purpose policy enforcement engine" for lack of a better phrase (perhaps thats a good enough summary) that allows granular decisions to be made based on "Documents" or "vDocuments" being evaluated using OPA's policy language Rego. I first started looking at this [project](https://www.openpolicyagent.org/) after playing around with K8s and trying to find a way to ensure my resources/yamls conformed to certain declarative conventions, as more teams move towards self-service systems for application developers and trust them to define their own policies.

From my research, I looked into how OPA was performing the validations against policies, what kind of traffic we could expect, and overall the flow of the request/decision from the application/resource and back.

The key idea behind OPA is to solve the problem presented by the following;

For IDENTITY *I*
Performing OPERATION *X*
On RESOURCE *Y*

##### The mechanics

I started with the idea that all I would do in this blogpost was identify the key components of OPA, write some of my thoughts about them and how effective they were... then I thought, why don't I just map out how OPA does what it does?! So, I found one of the examples on the [OPA website](https://www.openpolicyagent.org/docs/latest/ssh-and-sudo-authorization/), and deployed a [container](https://hub.docker.com/r/corfr/tcpdump/)(if you are savvy and don't trust Docker Hub - and good mindset to have - below is something you can build) to sniff the network traffic to understand what exactly is going on. Below I've sketched up how it looks based on what I know bout OPA;

![HTTP request flow](https://raw.githubusercontent.com/VariableExp0rt/VariableExp0rt.github.io/master/images/opa-diagram.png)

```
docker build -t tcpdump - <<EOF 
FROM ubuntu 
RUN apt-get update && apt-get install -y tcpdump 
ENTRYPOINT tcpdump
EOF
```

My suggestion here is to run through the tutorial until you reach the point of SSH-ing into the container and performing the "sudo ls /" command mentioned, but you can also see traffic some successful loading of policies and data too if you'd like.

`docker-compose -f <tutorial yaml> up &`

`docker run -it -v /path/to/storing/pcap/data:/data --net=container:<container id of OPA> corfr/tcpdump -i any -C <how many packets you want to capture> -w /data/dump01`

(Just pass arguments like `-i any` to the run command as it's already running tcpdump)

  1. First, you should see a `GET` request to OPA from the host requesting information about the needed context for the policy - this is defined in the pull package within that tutorial. This an important step, otherwise the module won't know what to gather to compare against when evaluating the policy.
  2. Second, the `POST` request sends additional information from the pam module including the service being requested i.e sudo, username etc, all this information was outlined by OPA in step 1.
  3. Third, a decision ID is returned along with whatever is returned as part of `allow` or `errors` which we defined in [] after the allow or errors "functions" (I say this because it looks so similar to Golang functions).
  
* This will be fairly unique to each integration, so you might find YMMW with others, which I have yet to test. You'll also see looking at the repository for this plugin that corroborates what we're seeing.

I think also monitoring/controlling who can load policies or data into OPA that is being evaluated is of course a good idea...

#### How does OPA enforce policies?

Simply, there is a very easy structure to follow in the rules that I have seen, the semantics probably aren't too important between "violation" or "deny", and these are suggested to get started with. Actually, this kind of reminds me of the implicit deny of network policies, you can set the default within these rules too (`default allow = false|true`) and we'll see how to do this, although there are countless policies/examples on the internet so I won't steal anyones thunder ;) .

By far the best place to start practicing with Rego and policies is the [Rego Playground](https://play.openpolicyagent.org/), basically similar to the Golang Playground (which is absolutely great too - and if you're into Golang, I'm sure you know it) so we'll head there and move on to writing some Rego. Before that though, make sure you have some data that you can test against first. In my previous post, I mentioned that my setup was a KIND cluster, you can spin this up fairly quickly and generate some deployments, network policies, any resource you'd like really, to get you familiar with with writing Rego;

`kubectl create <resource> <all your usual flags> --dry-run=client -o json`

IF you already have a bunch of yaml files with some interesting parameters then you can also use a project called *[yq](https://github.com/kislyuk/yq)* to convert between YAML and JSON like so (or just wrap this in a bash script - also below);

`cat <filename>.yaml | yq .`

```
#!/bin/sh

for i in /*.yaml; do
cat $i.yaml | yq . ;
done
```

Then just redirect the output when you execute the script ./<yoursimplescript>.sh >> newfile.json

You may want to separate it out by resource (Pods, Deployments, Network Policies) as this might be easier once we start giving the data to the OPA server.

#### Prototyping policies - developing rules

##### Rego playground

I tried to do my own example here, to get a feel for how the langauge is written and the way it should look (maybe there are some best practices I need to look for :| )

[To the playground!](https://play.openpolicyagent.org/p/X8iSYPF0Gs)

Put some of the manifests you converted earlier into the top right data section, write your policies, some simple allow and deny logic will be fine, and evaluate! I haven't done anything special either since this is my first try!

#### Testing your policies

##### Using the OPA binary and writing your own tests (a good way to learn more Rego, too)

OPA actually comes with it's own built-in test functionality, which is great if you're looking to automate testing of policies (which, often is the case, you need a separate framework for automating the testing of these, and is certainly true for the detection logic/content in SIEMs like Splunk...) and it is as simple as below;

`opa test file.rego file_test.rego` // here are my [examples](https://github.com/VariableExp0rt/openpolicyagent_resources)

I do need to understand the policy testing in further detail, so perhaps I can do a separate blog post on automating some of the testing around this.

I learnt a lot writing this blog post, and I think I will leave it for another to try out the policy within OPA gatekeeper and do a review of that too. Ciao!

(I always value feedback, so if there are any suggestions where I have got something wrong, any opportunity to learn is good!)
