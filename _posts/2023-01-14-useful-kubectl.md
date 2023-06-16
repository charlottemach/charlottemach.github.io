---
layout: post
title: "Useful kubectl commands"
date: 2023-01-14
tags: kubernetes hacks 
---

Sometimes I find myself going through old messages, command line history or slack search to find that one random command for debugging or fixing something. This is the very random collection of commands like that, as well as some that I just think are neat and should be better known.

## Finding all resources in a namespace

Namespace stuck in terminating? Sometimes not all the resources get cleaned up properly when deleting a namespace and you might end up with namespace that just won't go away.
This command get EVERYTHING in that namespace (contrary to `kubectl get all` which is limited to a subset of Kubernetes resources).
```
NAMESPACE=default # <- stuck namespace
kubectl api-resources --verbs=list --namespaced -o name  | \
  xargs -n 1 kubectl get --show-kind --ignore-not-found -n $NAMESPACE -oname | \
  xargs -n 1 kubectl delete -n $NAMESPACE
```
You can also add a dry-run flag to the delete or leave it out to just see the resources that are still there.


## Delete resources stuck in terminating

Sometimes other resources can get stuck in terminating too (especially when you have custom controllers that might be offline or not functioning as expected), when
something attaches a finalizers to a resource but doesn't finish cleanup.

Patching the object by removing the finalizer will delete it.
NOTE: This will not finish any steps that were supposed to be executed by whatever placed the finalizer, so run this only when you know what you're doing.
```
kubectl get <resource e.g. pods> -l <common label> -n <ns> -oname | \
  xargs kubectl patch  --type json --patch='[ { "op": "remove", "path": "/metadata/finalizers" } ]' -n <ns>
```

## Get old logs

Sometimes a container goes away too quickly to capture its logs, luckily you can get the previous container logs from a pod that has restarted as long as it is still around.
```
kubectl logs <podname> --previous
```
[K8s docs](https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_logs/)

## Ephemeral debug pods

I was so glad when this feature was introduced. No more random debug pods, no more changing your minimal image to use debugging tools in dev.
```
kubectl debug -it <pod> --image=busybox:1.28 --target=<pod>
```
[K8s docs](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container)

## Create a Kubernetes event manually

Either by applying the manifest
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Event
metadata:
  name: myevent
  namespace: default
type: Normal
message: 'some message'
involvedObject:
  kind: someObject
EOF
```
or by passing the JSON:
```
jq --arg now "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" -n '{apiVersion: "v1", kind: "Event", metadata: { namespace: "default", generateName: "event-" }, involvedObject: {apiVersion: "v1", kind: "Pod", name: "pod", namespace: "default", uid: "6d496a1a-bbc5-a5bb-12a3-133eq7ca0383", fieldPath: "spec.containers{container}"}, message: "Hello world!", firstTimestamp: $now, lastTimestamp: $now, type: "Normal", reason: "JustBecause", source: {component: "event-client"} }' | kubectl create -f -
```
which creates an event that can then be seen or used to for example trigger an operator's control loop manually.
```
$ kubectl get event
LAST SEEN   TYPE     REASON         OBJECT                            MESSAGE
4s          Normal   JustBecause    pod/pod                           Hello world!
```

