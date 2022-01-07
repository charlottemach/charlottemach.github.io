---
layout: post
title: "Getting started with Chaos Engineering using Litmus"
date: 2022-1-6
tags: litmus chaos kubernetes
---

The idea of using Chaos Engineering to improve systems is becoming more and more popular with the Kubernetes crowd, no doubt due to [Netflix's Simian Army](https://github.com/Netflix/SimianArmy) and the experiences that followed.
Simulating outages or different errors to test resilience, understanding your system better and battle-test your applications sounds like a good idea, here's one way of getting started.

## Understand the basics

Chaos Engineering is NOT about breaking things randomly and seeing what happens. It's about making an assumption about your system, trying to understand what would happen if a certain error or outage happens and testing that assumption in a controlled environment. For real-life systems that involves a large amount of preparation, communication and learning from whatever results a chaos experiment may have.

If you're just getting started and want to play around and learn, [Litmus](https://docs.litmuschaos.io/) is a good tool to try on Kubernetes.

## Install

Litmus describes itself as an end-to-end chaos engineering platform for cloud native infra and applications. It's control plane comes as a Kubernetes deployment consisting of a frontend, a server and a MongoDB backend, all together called the Chaos Center.

```
kubectl apply -f https://litmuschaos.github.io/litmus/2.4.0/litmus-2.4.0.yaml
```
The above command is all it takes to install it on your cluster (see the [docs](https://docs.litmuschaos.io/docs/getting-started/installation/) for the latest version).

Once the pods are up and running, you can access the frontend via `<IP>:<Nodeport>` which you can see in the service that gets created (in my case it's on a local cluster, so localhost:30711).
```
NAMESPACE     NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                       AGE
litmus        litmusportal-frontend-service   NodePort    10.98.60.168     <none>        9091:30711/TCP                                                66s
litmus        litmusportal-server-service     NodePort    10.109.185.162   <none>        9002:32132/TCP,8000:30885/TCP,9003:31270/TCP,3030:31009/TCP   65s
```

After logging in with the default credentials of user: admin and pw: litmus, you're prompted to change the password.
In the Chaos Center you can create workflows, add different hubs (like experiment libraries), agents  or monitor experiment runs and results.

(The 9002 port on the server lets you access the internal [GraphQL playground](https://github.com/graphql/graphql-playground).)


## Concepts

[This post](https://dev.to/litmus-chaos/how-litmus-orchestrates-chaos-3nnd) goes into more detail, but basically Litmus runs as an [operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/), using different CustomResources to define states and custom controllers to create Chaos Runners which execute the experiments as jobs.

##### CRs

* Chaos Experiment: the actual test you want to run
* Chaos Engine: the mapping of the test to the thing you want to test (infra/app)
* Chaos Result: giving you the current state and outcome of a test

A Chaos Workflow can be used to add additional steps around an experiment, for example to add preparation, cleanup or just to execute multiple operations.

## Running a test

With Litmus up and running you now have different options
* [Run a sample workflow using the UI](https://docs.litmuschaos.io/docs/getting-started/run-your-first-workflow)
* [Run a workflow from the ChaosHub](https://hub.litmuschaos.io/): The Hub contains different generic experiments like introducing network latency or node restarts, as well as application-specific experiments for a some cloud providers, Kafka, Cassandra.
* Write your own (using [Python](https://github.com/litmuschaos/litmus-python) or [Golang](https://github.com/litmuschaos/litmus-go))

For more resources, check out [awesome chaos engineering](https://github.com/dastergon/awesome-chaos-engineering) on GitHub.
