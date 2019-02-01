# Application Patterns for Containers in Kubernetes

## Sidecar Pattern
Sidecar design pattern is a form of single-node, multiple containers application patterns. Sidecar pattern advocates the usage of additional container for extending or enhancing the main container. Let's use an example of a web-server container that is based on legacy code and orginally served non-secure HTTP traffic. To add SSL capabilities a sidecar container can be added to extend the functionality of the original service. Some of the benefits of this pattern include the following:
* Separation of concerns: Let each container take care of their core concern. Web server container would be serving web pages while sidecar container would be processing server logs. This would be helpful during issues resolution stages when issues related to one concern should not conflict with other concern.
* Single responsibility principle: A container has primarily got one reason for change. In other words, each container do one thing well. Based on this principle, one can have separate teams work on different containers as their concerns are segregated well enough.
* Cohesiveness/Reusability: With sidecar containers for processing logs, this container can as well be reused at other places in the application.

### Exercise 1.1 - Deploy a Sidecar Container to enable HTTPS

#### Main application
For our example main application use a Nodejs Hello World service (evillgenius75/web-service)
#### Sidecar container 
To add https support I will use Nginx ssl proxy (ployst/nginx-ssl-proxy) container
#### Deployment
##### TLS/SSL keys
First we need to generate TLS certificate keys and add them to Kubernetes secrets. For that I am using script from nginx ssl proxy repository which combine all steps in one:
```console
git clone https://github.com/ployst/docker-nginx-ssl-proxy.git
cd docker-nginx-ssl-proxy
./setup-certs.sh /path/to/certs/folder
```

Adding TLS files to Kubernetes secrets

```
cd /path/to/certs/folder
kubectl create secret generic ssl-key-secret --from-file=proxykey=proxykey --from-file=proxycert=proxycert --from-file=dhparam=dhparam
```

##### Kubernetes sidecar deployment
In following configuration I have defined main application container “nodejs-hello” and nginx container “nginx”. Both containers run in the same pod and share pod resources, so in that way implementing sidecar pattern. One thing you want to modify is hostname, I am using not existing hostname appname.example.com for this example.

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: nodejs-hello
  labels:
    app: nodejs
    proxy: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs-hello
  template:
    metadata:
      labels:
        app: nodejs-hello
    spec:
      containers:
      - name: nodejs-hello
        image: beh01der/web-service-dockerized-example
        ports:
        - containerPort: 3000
      - name: nginx
        image: ployst/nginx-ssl-proxy
        env:
        - name: SERVER_NAME
          value: "appname.example.com"
        - name: ENABLE_SSL
          value: "true"
        - name: TARGET_SERVICE
          value: "localhost:3000"
        volumeMounts:
          - name: ssl-keys
            readOnly: true
            mountPath: "/etc/secrets"          
        ports:
        - containerPort: 80
          containerPort: 443
      volumes:
      - name: ssl-keys
        secret:
          secretName: ssl-key-secret
```


Save this file to deployment.yaml and create deployment Kubernetes object:

```
kubectl create -f deployment.yaml
```

Wait for pods to be Read:
```console
kubectl get pods

NAME                            READY     STATUS    RESTARTS   AGE
nodejs-hello-686bbff8d7-42mcn   2/2       Running   0          1m
```
#### Testing
For testing I setup two port forwarding rules. First is for application port and second for nginx HTTPS port:

```
kubectl -n test port-forward <pod> 8043:443
#and in new terminal window run
kubectl -n test port-forward <pod> 8030:3000
```

First lets validate that application respond on http and doesn’t respond on https requests


##### Using http
```console
curl -k -H "Host: appname.example.com" http://127.0.0.1:8030/ 
Hello World! 
I am undefined!
```
##### Using https
```console
curl -k -H "Host: appname.example.com" https://127.0.0.1:8030/ 
curl: (35) Server aborted the SSL handshake
```

>**Note:** SSL handshake issue is expected as our “legacy” application doesn’t support https and even if it would it must serve https connection on different port than http. The test goal was to demonstrate the response.

Time to test connection through sidecar nginx ssl proxy

```console
curl -k  -H "Host: appname.example.com" https://127.0.0.1:8043/
Hello World!
I am undefined!
```

Great! We have got expected output through https connection.

## Ambassador Pattern
Ambassador pattern is another form of single-node, multiple containers application patterns. This pattern advocates usage of additional containers as proxies to the external group/cluster of servers. The primary goal is to simplify the access of external servers using the ambassador container. This container can be grouped as a logical atomic unit such that the application container can invoke this ambassador/proxy container which can, then, invoke the cluster of servers. The following is primary advantage of using ambassador container:

* Let ambassador container takes care of connecting to cluster of servers; Application container connects the ambassador container using “localhost” paradigm.
* Above enables using standalone server (and not cluster of servers) in the development environment. Application container opens the connection to the server on “localhost” and find the proxy without any service discovery.

## Adapter Pattern
Adapter pattern is yet another form of single-node, multiple containers application patterns. It advocates usage of additional containers to present a simplified, homogenized view of an application running within a container. These containers can be called as adapter container. The main container communicates with the adapter container through localhost or a shared local volume. This is unlike ambassador container which simply has access to external servers.

A concrete example of the adapter pattern is adapters that ensure all containers in a system have the same monitoring interface. 