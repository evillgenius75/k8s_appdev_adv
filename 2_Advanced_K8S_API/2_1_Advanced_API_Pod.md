# Using Advanced POD API features

## Liveness and Readiness Probes
Many containers, especially web services, won't have an exit code that accurately tells Kubernetes whether the container was successful. Most web servers won't terminate at all! How do you tell Kubernetes about your container's health? You define container probes.
Container probes are small processes that run periodically. The result of this process determines Kubernetes' view of the container's state—the result of the probe is one of Success, Failed, or Unknown.
You will most often use container probes through Liveness and Readiness probes. Liveness probes are responsible for determining if a container is running or when it needs to be restarted. Readiness probes indicate that a container is ready to accept traffic. Once all of its containers indicate they are ready to accept traffic, the pod containing them can accept requests.
There are three ways to implement these probes. One way is to use HTTP requests, which look for a successful status code in response to making a request to a defined endpoint. Another method is to use TCP sockets, which returns a failed status if the TCP connection cannot be established. The final, most flexible, way is to define a custom command, whose exit code determines whether the check is successful.

### Readiness Probes
In this example, we've got a simple web server that starts serving traffic after some delay while the application starts up. If we didn't configure our readiness probe, Kubernetes would either start sending traffic to the container before we're ready, or it would mark the pod as unhealthy and never send it traffic.

#### Exercise 2.1 - Readiness Probe
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

```console
kubectl create -f liveready-pod.yaml
```

Wait for pods to be Ready:
```console
kubectl get pods
```

```output
NAME                            READY     STATUS    RESTARTS   AGE
liveness-readiness-pod   1/1     Running   0          19s
```
##### Testing

Describe the pod to see the events. Kubernetes thinks the pod is unhealthy. In reality, we just have an application that takes a little while to boot up!

```console
kubectl describe pod liveness-readiness-pod
```

```output
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

### Liveness Probes
Another useful way to customize when Kubernetes restarts your applications is through liveness probes. Kubernetes will execute a container's liveness probe periodically to check that the container is running correctly. If the probe reports failure, the container is restarted. Otherwise, Kubernetes leaves the container as-is.

#### Lab 2.2 Liveness and Readiness Probes combined
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

```console
kubectl delete pod liveness-readiness-pod
kubectl apply -f liveready-pod.yaml
```

## Pod Security
A little-used, but powerful feature of Kubernetes pods are security context, an object that configures roles and privileges for containers. A security context can be defined at the pod level or at the container level. If the container doesn't declare its own security context, it will inherit from the parent pod.

Security contexts generally configure two fields: the user ID that should be used to run the pod or container, and the group ID that should be used for filesystem access. These options are useful when modifying files in a volume that is mounted and shared between containers, or in a persistent volume that's shared between pods. We'll cover volumes in detail in a coming chapter, but for now think of them like a regular directory on your computer's filesystem.

To illustrate, let's create a pod with a container that writes the current date to a file in a mounted volume every five seconds.

### Lab 2.3 - Pod Security
Create a pod that defines the runAsUser and the fsGroup ID for all of the containers in the pod

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: security-context-pod
spec:
  securityContext:
    # User ID that containers in this pod should use
    runAsUser: 45
    # Group ID for filesystem access
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
Save this file to security-pod.yaml and create pod Kubernetes object:

```console
kubectl create -f security-pod.yaml.yaml
```

Wait for pods to be Ready:
```console
kubectl get pods
```

```output
NAME                            READY     STATUS    RESTARTS   AGE
security-context-pod    1/1     Running   0          19s
```

#### Testing
You will need to exec into the pod with an `ls` of the directory and see the properties of the file where the user is UID45 and the FileSystem Group is 213
```console
kubectl exec security-context-pod -- ls -l /etc/directory
```

```output
total 4
-rw-r--r--    1 45       231             58 Feb  3 19:07 file.txt
```

## Configuration through ConfigMaps and Secrets

### ConfigMaps

Many applications require configuration via some combination of config files, command line arguments, and environment variables. These configuration artifacts should be decoupled from image content in order to keep containerized applications portable. The ConfigMap API resource provides mechanisms to inject containers with configuration data while keeping containers agnostic of Kubernetes. ConfigMaps can be used to store fine-grained information like individual properties or coarse-grained information like entire config files or JSON blobs.

The ConfigMap API resource holds key-value pairs of configuration data that can be consumed in pods or used to store configuration data for system components such as controllers. ConfigMap is similar to Secrets, but designed to more conveniently support working with strings that do not contain sensitive information.

There are 2 methods in which a pod can consume a ConfigMap. 
1. Using a ConfigMap through a volume:
* Once created, add the ConfigMap as a volume in your pod's specification. Then, mount that volume into the container's filesystem. Each property name in the ConfigMap will become a new file in the mounted directory, and the contents of each file will be the value specified in the ConfigMap.
2. Using a ConfigMap through environment variables:
* Another useful way of consuming a ConfigMap you have created in your Kubernetes cluster is by injecting its data as environment variables into your container.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: pod-env-var
spec:
  containers:
    - name: env-var-configmap
      image: nginx:1.7.9
      envFrom:
        - configMapRef:
          name: example-configmap
```


>**Note:** ConfigMaps are not intended to act as a replacement for a properties file. ConfigMaps are intended to act as a reference to multiple properties files. 

ConfigMaps must be created before they are consumed in pods unless they are marked as optional. References to ConfigMaps that do not exist will prevent the pod from starting. Controllers may be written to tolerate missing configuration data; consult individual components configured via ConfigMap on a case-by-case basis.

When a ConfigMap already being consumed in a volume is updated, projected keys are eventually updated as well. Kubelet is checking whether the mounted ConfigMap is fresh on every periodic sync. However, it is using its local ttl-based cache for getting the current value of the ConfigMap. As a result, the total delay from the moment when the ConfigMap is updated to the moment when new keys are projected to the pod can be as long as kubelet sync period + ttl of ConfigMaps cache in kubelet.
>**Note:** A container using a ConfigMap as a subPath volume will not receive ConfigMap updates.

References via configMapKeyRef to keys that do not exist in a named ConfigMap will prevent the pod from starting.

ConfigMaps used to populate environment variables via envFrom that have keys that are considered invalid environment variable names will have those keys skipped. The pod will be allowed to start. There will be an event whose reason is InvalidVariableNames and the message will contain the list of invalid keys that were skipped.

#### Lab 2.4 - ConfigMap for Redis Cache

Let's us a look at a real-world example: configuring redis using ConfigMap. We want to inject redis with the recommended configuration for using redis as a cache. 

First create a file called redis-config:
```console
mkdir config
cd config
cat > redis-config << EOF
maxmemory 2mb
maxmemory-policy allkeys-lru
EOF
```

Create a ConfigMap from the file:
```console
kubectl create configmap redis-config --from-file=redis-config
```

```console
kubectl describe configmap redis-config
```

```output
Name:         redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb
maxmemory-policy allkeys-lru

Events:  <none>
```
Now we will create a Redis pod that will use the redis-config to configure the redis container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: kubernetes/redis:v1
    env:
    - name: MASTER
      value: "true"
    ports:
    - containerPort: 6379
    resources:
      limits:
        cpu: "0.1"
    volumeMounts:
    - mountPath: /redis-master-data
      name: data
    - mountPath: /redis-master
      name: config
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: redis-config
        items:
        - key: redis-config
          path: redis.conf
```
Notice that this pod has a ConfigMap volume that places the redis-config key of the example-redis-config ConfigMap into a file called redis.conf. This volume is mounted into the /redis-master directory in the redis container, placing our config file at /redis-master/redis.conf, which is where the image looks for the redis config file for the master.

Save this file to secret-mysql.yaml and create pod Kubernetes object:

```console
kubectl create -f secret-mysql.yaml
```

Wait for pods to be Ready:
```console
kubectl get pods
```

```output
NAME                            READY     STATUS    RESTARTS   AGE
redis                          1/1     Running   0          19s
```

##### Testing

Use `kubectl exec` to enter the pod and run the `redis-cli` tool to verify that the configuration was correctly applied:

```console
kubectl exec -it redis redis-cli
```

```output
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "2097152"
127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "allkeys-lru"
127.0.0.1:6379> exit
```
### Secrets

If you understand creating and consuming ConfigMaps, you also understand how to use Secrets. The primary difference between the two is that Secrets are designed to hold sensitive data—things like keys, tokens, and passwords.

Kubernetes will avoid writing Secrets to disk—instead using tmpfs volumes on each node, so they can't be left behind on a node. However, etcd, Kubernetes’ configuration key-value store, stores secrets in plaintext. Further, if there are multiple replicas of etcd, data is not necessarily replicated over SSL. Make sure your administrator restricts etcd access before using Secrets.

Kubernetes will only send Secrets to a node when one of the node's pods explicitly requires it, and removes the Secret if that pod is removed. Plus, a Secret is only ever visible inside the Pod that requested it.

When adding Secrets via YAML, they must be encoded in base64. base64 is not an encryption method and does not provide any security for what is encoded—it's simply a way of presenting a string in a different format. Do not commit your base64- encoded secrets to source control or share them publicly.

#### Lab 2.5 - Configure mySQL with a Secret for the DB Password

First we need to configure the secret which can be done in a few different ways. The quickest way for single string value pairs is using the `--from-literal`flag of the `kubectl create seceret` command.

```console
kubectl create secret generic mysql-password --from-literal=password=MySuperSecretPassword
```

Verify the secret was created and the actual password is not visible

```console
kubectl describe secret mysql-password
```

```output
Name:         mysql-password
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  21 bytes
```

Now deploy a mySQL pod that consumes the secret as an environment variable:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-password
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        emptyDir: {}
```
The preceding YAML file creates a service that allows other Pods in the cluster to access the database. The Service option clusterIP: None lets the Service DNS name resolve directly to the Pod’s IP address.

Save this file to secret-mysql.yaml and create pod Kubernetes object:

```console
kubectl create -f secret-mysql.yaml
```

Wait for pods to be Ready:
```console
kubectl get pods
```

```output
NAME                    READY   STATUS    RESTARTS   AGE
mysql-d4b54994b-7drd6   1/1     Running   0          15m
```

##### Testing

Run a MySQL client to connect to the server:

```console
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -pMySuperSecretPassword
```

This command creates a new Pod in the cluster running a MySQL client and connects it to the server through the Service. If it connects, you know your stateful MySQL database is up and running.

```output
If you don't see a command prompt, try pressing enter.

mysql>
```

