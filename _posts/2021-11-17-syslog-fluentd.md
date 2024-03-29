---
layout: post
title: "Adding a secondary fluentd to Openshift Logging as a syslog receiver"
date: 2021-11-17
tags: OpenShift fluentd logging EFK syslog
---

OpenShift comes with loads of handy operators managing various Kubernetes resources for you, some open source, some not.
[OpenShift Logging](https://github.com/openshift/cluster-logging-operator) is the wrapper for an EFK stack running on your cluster.
Like with other operators you don't create your own daemonsets or deployments, instead you configure it with a CustomResource called
[ClusterLogging](https://docs.openshift.com/container-platform/4.9/logging/cluster-logging.html).

So if you're thinking of using this Fluentd for anything other than logs from this cluster, you're pretty much out of luck.

Note: You can set everything to Unmanaged and change the configs, but that's not sustainable as you're basically disabling any operator actions. Also editing what's supposed to be an indented file represented as one long string is no fun.

You can however piggyback on the resources deployed by the operator by adding a secondary Fluentd in the same namespace with some modifications.
We'll create a Fluentd that acts as a syslog receiver, though you could in theory add any input you like, see the list of [Fluentd input plugins](https://www.fluentd.org/plugins).
And we'll re-use the ElasticSearch and Kibana pods to visualize those logs.

### Secondary Fluentd

Assuming the logging operator is installed and one instance of the EFK pods is up and running:

Let's start with a ConfigMap, where we add the Fluentd config file receiving logs on (the pods) port 5140, and sending everything as is to the local ElasticSearch.
The exact paths might vary, but can be taken from the primary Fluentd daemonset.
```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: openshift-logging
data:
  fluent.conf: |-
    #syslog plugin
    <source>
      @type udp
      port 5140
      bind 0.0.0.0
      <parse>
        @type none
      </parse>
      tag system
    </source>
    <match **>
      @type copy
      <store>
        @type elasticsearch
        host elasticsearch.openshift-logging.svc
        port 9200
        logstash_format true
        scheme https
        ssl_version TLSv1_2
        client_key '/var/run/ocp-collector/secrets/fluentd/tls.key'
        client_cert '/var/run/ocp-collector/secrets/fluentd/tls.crt'
        ca_file '/var/run/ocp-collector/secrets/fluentd/ca-bundle.crt'
      </store>
    </match>
```
There is a syslog input plugin as well, but that might need specific settings depending on the syslog message format.
After the config, we can create our deployment (a daemonset might be more useful for you).
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: fluentd-syslog
  name: fluentd-syslog
  namespace: openshift-logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fluentd-syslog
  template:
    metadata:
      labels:
        app: fluentd-syslog
    spec:
      containers:
      - image: quay.io/fluentd_elasticsearch/fluentd:v3.3.0
        imagePullPolicy: IfNotPresent
        name: fluentd
        env:
        - name: FLUENTD_ARGS
          value: --no-supervisor
        ports:
        - containerPort: 24224
          name: forward
          protocol: TCP
        - containerPort: 5140
          name: syslog
          protocol: TCP
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: false
        - name: config
          mountPath: /etc/fluent/config.d
        - mountPath: /var/run/ocp-collector/secrets/fluentd
          name: fluentd
          readOnly: true
        - mountPath: /etc/pki/ca-trust/extracted/pem/
          name: fluentd-trusted-ca-bundle
          readOnly: true
      volumes:
      - name: varlog
        emptydir: {}
      - configMap:
          defaultMode: 420
          name: fluentd-config
        name: config
      - name: fluentd
        secret:
          defaultMode: 420
          secretName: fluentd
      - configMap:
          defaultMode: 420
          items:
          - key: ca-bundle.crt
            path: tls-ca-bundle.pem
          name: fluentd-trusted-ca-bundle
        name: fluentd-trusted-ca-bundle
```
The only differences between this an a generic deployment are the additional mounted volumes for the certificates as well as an image that already contains the ElasticSearch plugin.

You can build your own image as well, see the README of the original [Fluentd image](https://hub.docker.com/r/fluent/fluentd/) for instructions.

After creating the deployment we only need to expose the port from inside the pod to the outside world. If you have a loadbalancer or ingress, use that. Otherwise a Nodeport works just as well.
```
---
apiVersion: v1
kind: Service
metadata:
  name: fluentd-syslog
  namespace: openshift-logging
  labels:
    app: fluentd-syslog
spec:
  type: NodePort
  ports:
  - port: 5140
    targetPort: 5140
    nodePort: 30514
    protocol: UDP
  selector:
    app: fluentd-syslog
```
This service exposed the deployment on the nodes port 30514.

### Testing

Now you can test the deployment by sending a message yourself, for example
```
nc -w0 -u <Node IP> 30514 <<< "FAKE SYSLOG MESSAGE"
```
After a few seconds the log should show up inside your Kibana.
