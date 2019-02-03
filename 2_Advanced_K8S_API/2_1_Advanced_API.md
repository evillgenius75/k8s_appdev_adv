# Using Advanced POD API features

## Liveness and Readiness Probes
Many containers, especially web services, won't have an exit code that accurately tells Kubernetes whether the container was successful. Most web servers won't terminate at all! How do you tell Kubernetes about your container's health? You define container probes.
Container probes are small processes that run periodically. The result of this process determines Kubernetes' view of the container's state—the result of the probe is one of Success, Failed, or Unknown.
You will most often use container probes through Liveness and Readiness probes. Liveness probes are responsible for determining if a container is running or when it needs to be restarted. Readiness probes indicate that a container is ready to accept traffic. Once all of its containers indicate they are ready to accept traffic, the pod containing them can accept requests.
There are three ways to implement these probes. One way is to use HTTP requests, which look for a successful status code in response to making a request to a defined endpoint. Another method is to use TCP sockets, which returns a failed status if the TCP connection cannot be established. The final, most flexible, way is to define a custom command, whose exit code determines whether the check is successful.

## Readiness Probes
In this example, we've got a simple web server that starts serving traffic after some delay while the application starts up. If we didn't configure our readiness probe, Kubernetes would either start sending traffic to the container before we're ready, or it would mark the pod as unhealthy and never send it traffic.

Let's start with a readiness probe that makes a request to http://
localhost:8000/ after five seconds, and only looks for one failure before it marks the pod as unhealthy.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: liveness-readiness-pod
spec:
  containers:
    - name: server
      image: python:2.7-alpine
      # Check that a container is ready to handle requests.
      readinessProbe:
        # After thirty seconds, check for a 200 response on localhost:8000/
        # and check four times before thinking we've failed.
        initialDelaySeconds: 5 # Update this to 30
        failureThreshold: 1     # Update this to 4
        httpGet:
          path: /
          port: 8000
      # This container starts a simple web server after 45 seconds.
      env:
        - name: DELAY_START
          value: "45"
      command: ["/bin/sh"]
      args: ["-c", "echo 'Sleeping...'; sleep $(DELAY_START); echo 'Starting server...'; python -m SimpleHTTPServer"]
```
Save this file to liveready-pod.yaml and create pod Kubernetes object:

```
kubectl create -f liveready-pod.yaml
```

Wait for pods to be Ready:
```console
kubectl get pods

NAME                            READY     STATUS    RESTARTS   AGE
liveness-readiness-pod   1/1     Running   0          19s
```
## Testing

Describe the pod to see the events. Kubernetes thinks the pod is unhealthy. In reality, we just have an application that takes a little while to boot up!

```console
kubectl describe pod liveness-readiness-pod 
....
Events:
  Type     Reason     Age                From                               Message
  ----     ------     ----               ----                               -------
  Normal   Scheduled  13m                default-scheduler                  Successfully assigned default/liveness-readiness-pod to aks-agentpool-22325591-2
  Normal   Pulling    13m                kubelet, aks-agentpool-22325591-2  pulling image "python:2.7-alpine"
  Normal   Pulled     13m                kubelet, aks-agentpool-22325591-2  Successfully pulled image "python:2.7-alpine"
  Normal   Created    13m                kubelet, aks-agentpool-22325591-2  Created container
  Normal   Started    13m                kubelet, aks-agentpool-22325591-2  Started container
  Warning  Unhealthy  12m (x4 over 13m)  kubelet, aks-agentpool-22325591-2  Readiness probe failed: Get http://172.22.0.37:8000/: dial tcp 172.22.0.37:8000: connect: connection refused
  ```

## Liveness Probes
Another useful way to customize when Kubernetes restarts your applications is through liveness probes. Kubernetes will execute a container's liveness probe periodically to check that the container is running correctly. If the probe reports failure, the container is restarted. Otherwise, Kubernetes leaves the container as-is.

Let's build off the previous example and add a liveness probe for our web server container. Instead of using an httpGet request to probe the container, this time we'll use a command whose exit code indicates whether the container is dead or alive.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: liveness-readiness-pod
spec:
  containers:
    - name: server
      image: python:2.7-alpine

      # Check that a container is ready to handle requests.
      readinessProbe:
        # After thirty seconds, check for a 200 response on localhost:8000/
        # and check four times before thinking we've failed.
        initialDelaySeconds: 30 # Update this to 30
        failureThreshold: 4     # Update this to 4
        httpGet:
          path: /
          port: 8000

      # Check that a container is still alive and running
      livenessProbe:
        # After sixty seconds, start checking periodically whether we have
        # and index.html file in our web server's directory. 
        # Wait five seconds before repeating the check. 
        # This file will never exist, so our container gets restarted.
        initialDelaySeconds: 60
        periodSeconds: 5
        exec:
          command: ["ls", "index.html"]

      # This container starts a simple web server after 45 seconds.
      env:
        - name: DELAY_START
          value: "45"
      command: ["/bin/sh"]
      args: ["-c", "echo 'Sleeping...'; sleep $(DELAY_START); echo 'Starting server...'; python -m SimpleHTTPServer"]
```
Now, delete and recreate the new pod. Then, inspect its event log just as above. We'll see that our changes to the readiness probe have worked, but then the liveness probe finds that the container doesn't have the required file so the container will be killed and restarted.

## Pod Security
A little-used but powerful feature of Kubernetes pods is that you can declare its security context, an object that configures roles and privileges for containers. A security context can be defined at the pod level or at the container level. If the container doesn't declare its own security context, it will inherit from the parent pod.

Security contexts generally configure two fields: the user ID that should be used to run the pod or container, and the group ID that should be used for filesystem access. These options are useful when modifying files in a volume that is mounted and shared between containers, or in a persistent volume that's shared between pods. We'll cover volumes in detail in a coming chapter, but for now think of them like a regular directory on your computer's filesystem.

To illustrate, let's create a pod with a container that writes the current date to a file in a mounted volume every five seconds.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: security-context-pod
spec:
  securityContext:
    # User ID that containers in this pod should use
    runAsUser: 45
    # Gropu ID for filesystem access
    fsGroup: 231
  volumes:
    - name: simple-directory
      emptyDir: {}
  containers:
    - name: example-container
      image: alpine
      command: ["/bin/sh"]
      args: ["-c", "while true; do date >> /etc/directory/file.txt; sleep 5; done"]
      volumeMounts:
        - name: simple-directory
          mountPath: /etc/directory
```

## Resource assignments

# Using Advanced Deployment API Features

## Update Strategy

## 