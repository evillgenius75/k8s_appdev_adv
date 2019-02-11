# Exercise 1.2 - Deploy a Sidecar Container to enable HTTPS

## Main application
For our example main application use a Nodejs Hello World service (evillgenius75/web-service)
## Sidecar container 
To add https support I will use Nginx ssl proxy (ployst/nginx-ssl-proxy) container
## Deployment
### TLS/SSL keys
First we need to generate TLS certificate keys and add them to Kubernetes secrets. For that use a script from nginx ssl proxy repository which combines all steps in one:
```console
git clone https://github.com/ployst/docker-nginx-ssl-proxy.git
cd docker-nginx-ssl-proxy
./setup-certs.sh /path/to/certs/folder
```

Adding TLS files to Kubernetes secrets

```console
cd /path/to/certs/folder
kubectl create secret generic ssl-key-secret --from-file=proxykey=proxykey --from-file=proxycert=proxycert --from-file=dhparam=dhparam
```

### Kubernetes sidecar deployment
The following configuration defines the main application containers, “nodejs-hello” and nginx container “nginx”. Both containers run in the same pod and share pod resources and serve as an example implementation of the sidecar pattern. One thing you want to modify is hostname, you can us a non existing hostname appname.example.com for this example.

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: sidecar
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
        image: evillgenius/nodehello:v1
        ports:
        - containerPort: 3000
      - name: nginx
        image: ployst/nginx-ssl-proxy
        env:
        - name: SERVER_NAME
          value: "www.evillgenius.com"
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


Save this file to sidecar-ex.yaml and create deployment Kubernetes object:

```console
kubectl create -f sidecar-ex.yaml
```

Wait for pods to be Ready:
```console
kubectl get pods

NAME                            READY     STATUS    RESTARTS   AGE
sidecar-686bbff8d7-42mcn   2/2       Running   0          1m
```
## Testing
For testing, setup two port forwarding rules. First is for application port and second for nginx HTTPS port:

```console
kubectl port-forward <pod> 8043:443 &
```

In a new terminal window run:
```console
kubectl port-forward <pod> 8030:3000 &
```

First lets validate that application responds on http and doesn’t respond on https requests


### Using http
```console
curl -k -H "Host: appname.example.com" http://127.0.0.1:8030/ 
Hello World! 
I am undefined!
```
### Using https
```console
curl -k -H "Host: appname.example.com" https://127.0.0.1:8030/ 
curl: (35) LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to 127.0.0.1:8030
```

>**Note:** The SSL handshake issue is expected as our “legacy” application doesn’t support https and even if SSL was configured, another port would have to be configured to serve https. The test's goal was to demonstrate the response.

Time to test connection through sidecar nginx ssl proxy

```console
curl -k  -H "Host: appname.example.com" https://127.0.0.1:8043/
Hello World!
I am undefined!
```

### Cleanup
Find and kill the portforward kubectl processes.
```console
ps | grep kubectl
kill <pids of kubectl processes>
```
Great! We have got expected output through https connection.

