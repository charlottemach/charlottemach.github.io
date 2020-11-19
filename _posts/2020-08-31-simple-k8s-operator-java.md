---
layout: post
title: "Writing a simple K8s Operator in Java"
date: 2020-08-31
tags: java kubernetes operator
---

Kubernetes Operator are often used to simplify the usage of applications or software in and outside a K8s cluster. They allow you to extend K8s by adding custom controllers for custom resources, allowing for example simpler database upgrades, application maintenance and automated creation of K8s resources.

There is a large number of Operators readily availabe, from [OperatorHub](https://operatorhub.io/) or [GitHub](https://github.com/operator-framework/awesome-operators).

### Why write your own?

You might be providing an application that needs certain maintenance that you want to automate. You might be providing a database to your developers that you don't want them to have to provision themselves. You might just wanna learn more about Kubernetes.

### Getting started

1. Create a simple Java application in your favourite IDE (e.g. [this how-to for IntelliJJ](https://www.jetbrains.com/help/idea/creating-and-running-your-first-java-application.html)).

2. Add Maven support ([beginner's guide](https://www.jetbrains.com/help/idea/convert-a-regular-project-into-a-maven-project.html#develop_with_maven)).

3. Add the Maven dependency for the [java-operator-sdk](https://github.com/ContainerSolutions/java-operator-sdk/blob/master/README.md).

    ```xml
    <dependency>
      <groupId>com.github.containersolutions</groupId>
      <artifactId>operator-framework</artifactId>
      <version>{see https://search.maven.org/search?q=a:operator-framework for latest version}</version>
    </dependency>
    ```

4. Create a CRD.yaml for your application ([example](https://github.com/ContainerSolutions/java-operator-sdk/blob/master/samples/webserver/crd/crd.yaml)).

5. Create a Controller class with two methods (`deleteResource` and `createOrUpdateResource`).
This is the code being run when the Kubernetes API Server sends events about your custom resource.

6. Create a POJO representation for your CRD, [see samples with spec and status here](https://github.com/ContainerSolutions/java-operator-sdk/tree/master/samples/webserver/src/main/java/com/github/containersolutions/operator/sample).

7. Register your controller in your main class. 

    ```java
    public class YourOperator {

       public static void main(String[] args) {
           Operator operator = new Operator(new DefaultKubernetesClient());
           operator.registerController(new YourController());
       }
    }
    ```
8. Before running your code, make sure your local `kubectl` points to a cluster, for example if `kubectl version` gives no errors. If not check how to [set it up](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

9. Package your code into a jar ([beginner's guide](https://www.jetbrains.com/help/idea/creating-and-running-your-first-java-application.html#package)).

10. Create a Dockerfile for your Operator ([example](https://github.com/ContainerSolutions/java-operator-sdk/blob/master/samples/webserver/Dockerfile)).

    ```Dockerfile
    FROM openjdk:12-alpine

    ARG JAR_FILE
    ADD target/${JAR_FILE} /usr/share/operator/your-operator.jar

    ENTRYPOINT ["java", "-jar", "/usr/share/operator/your-operator.jar"]
    ```
11. Create your CRD and deploy your Operator to the cluster ([example for deployment YAML](https://github.com/ContainerSolutions/java-operator-sdk/blob/master/samples/webserver/k8s/deployment.yaml)).

    ```bash
    kubectl apply -f CRD.yaml
    kubectl apply -f deployment.yaml
    ```
12. Create your CustomResource, matching your CRD and watch how the Operator reacts to the creation of the CustomResource!


### Further reading

1. [Operator Pattern: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
2. [More on CustomResources: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
