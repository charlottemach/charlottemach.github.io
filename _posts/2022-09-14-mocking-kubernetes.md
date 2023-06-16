---
layout: post
title: "Testing: Mocking the Kubernetes API in Go"
date: 2022-09-14
tags: kubernetes testing golang 
---

Whether you're writing an operator just interacting with the Kubernetes API for other reasons, you'll want to write unit tests for your code. Luckily the Kubernetes Go-client comes with it's [own mocking library](https://pkg.go.dev/k8s.io/client-go/kubernetes/fake) to fake any reaction your API server might have.

## Getting started

For a simple mock like creating a deployment we can start by creating a fake ClientSet and swapping out the dependency.

For a function that creates a deployment, for example
```
func createDeployment(ctx context.Context, client kubernetes.Interface, name string, namespace string) error { ... }
```
we can create a simple `main_test.go` (or whatever your package is called) that looks like this:
```
package main

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/require"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes/fake"
)

func TestCreateDeployment(t *testing.T) {
    tests := []struct {
        name string
        err  error

        client *fake.Clientset
    }{
        {
            name:   "success - created deployment",
            err:    nil,
            client: fake.NewSimpleClientset(),
        },
    }
    for _, test := range tests {
        t.Run(test.name, func(t *testing.T) {
            ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
            defer cancel()

            err := createDeployment(ctx, test.client, "test", "default")
            if test.err == nil {
                require.NoError(t, err)
            } else {
                require.EqualError(t, err, test.err.Error())
            }
        })
    }
}

```
We've replaced the client with the fake client, so now calling the CreateDeployment function has a simple happy path. But what if we want to ensure our code works as intended if the deployment doesn't get created?

## Adding a Failure

We can add a test case for an error:
```
        {
            name:   "failure - creation errors",
            err:    fmt.Errof("error"),
            client: fake.NewSimpleClientset(),
        },
```
but that doesn't actually tell the fake client to return anything else than before. What we need is something called a reactor, which (as the name implies) can add reactions to requests that are sent to the fake API server.

So we add a field to our tests struct, for adding a reactor function:
```
    reactor func(t *testing.T, client *fake.Clientset) func(action k8sTesting.Action) (bool, runtime.Object, error)
```
and change the failure case to:
```
        {
            name:    "failure - creation errors",
            err:     fmt.Errorf("fake error"),
            client:  fake.NewSimpleClientset(),
            reactor: createDeploymentFail,
        },
```
Now we can create a reaction function:
```
func createDeploymentFail(t *testing.T, client *fake.Clientset) func(action k8sTesting.Action) (bool, runtime.Object, error) {
    return func(action k8sTesting.Action) (bool, runtime.Object, error) {
        return true, nil, fmt.Errorf("fake error")
    }
}

```
and don't forget to add the reactor to the client in the test run:
```
test.client.PrependReactor("create", "deployments", test.reactor(t, test.client))
```
This lets us control the return values of any request, but it can easily make you want to add functions for all the "common" behaviour you'd expect from the Kubernetes API. Like creating a deployment would mean there's now a replicaset and pods and they're up and running, right?

But since the point of this is not to emulate or rebuild Kubernetes functionality it comes down to figuring out what resources your own code is relying on and is reacting to. It's also interesting to see if the reaction to your cluster misbehaving are as expected.

You can also add resources to the fake client immediately during initialization, e.g. 
```
client: fake.NewSimpleClientset(
                &appsv1.Deployment{
                    ObjectMeta: metav1.ObjectMeta{
                        Name:      "test",
                        Namespace: "default",
                        Labels: map[string]string{
                            "app": "test",
                        },
                    },
                    Spec: appsv1.DeploymentSpec{
                        Replicas: Pointer(int32(2)),
                        Selector: &metav1.LabelSelector{
                            MatchLabels: map[string]string{
                                "app": "test",
                            },
                        },
                        Template: corev1.PodTemplateSpec{
                            ObjectMeta: metav1.ObjectMeta{
                                Labels: map[string]string{
                                    "app": "test",
                                },
                            },
                        },
                    },
                },
            ),
```
This can make it harder too read after adding a few resources, so focussing on only the absolute minimal examples or even reading a manifest file into an object might be preferable.

Creating a deployment is just a very simple example of what can be tested using the mocking feature of the go-client library, but it already requires some knowledge of Kubernetes internals like which resources exist and what they look like, what actions can and need to be mocked. Some simple tests for success and error responses are a good way to get started and see that unit testing your Go code that touches Kubernetes is not such a complicated task after all.
