# Kubernetes Metrics Architecture

## Metrics Server

Starting from Kubernetes 1.8, resource usage metrics, such as container CPU and memory usage, are available in Kubernetes through the Metrics API. These metrics can be either accessed directly by user, for example by using kubectl top command, or used by a controller in the cluster, e.g. Horizontal Pod Autoscaler, to make decisions.

Through the Metrics API you can get the amount of resource currently used by a given node or a given pod. This API doesn’t store the metric values, so it’s not possible for example to get the amount of resources used by a given node 10 minutes ago.

The API is no different from any other API:

it is discoverable through the same endpoint as the other Kubernetes APIs under /apis/metrics.k8s.io/ path
it offers the same security, scalability and reliability guarantees

Metrics Server is a cluster-wide aggregator of resource usage data. Starting from Kubernetes 1.8 it’s deployed by default in clusters as a Deployment object. 

Metric server collects metrics from the Summary API, exposed by Kubelet on each node.

Metrics Server registered in the main API server through Kubernetes aggregator.

## Horizontal Pod Autoscaler

The Horizontal Pod Autoscaler automatically scales the number of pods in a replication controller, deployment or replica set based on observed CPU utilization (or, with custom metrics support, on some other application-provided metrics). Note that Horizontal Pod Autoscaling does not apply to objects that can’t be scaled, for example, DaemonSets.

The Horizontal Pod Autoscaler is implemented as a Kubernetes API resource and a controller. The resource determines the behavior of the controller. The controller periodically adjusts the number of replicas in a replication controller or deployment to match the observed average CPU utilization to the target specified by user.

## Lab 3.1 Implement an autoscale based on CPU metric


```yaml
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: php-apache
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  labels:
    name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      name: php-apache
  template:
    metadata:
      labels:
        name: php-apache
    spec:
      containers:
      - image: k8s.gcr.io/hpa-example
        name: php-apache
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
```

