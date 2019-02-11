# Exercise 1.2 - Deploy a Sidecar Container to enable HTTPS

## Main application
For our example main application use an echo service that outputs some Lorem text (evillgenius75/echo-server)
## Sidecar container 
To add https support I will use Nginx ssl proxy (evillgenius/tls-sidecar) container
## Deployment
### TLS/SSL keys
First we need to generate TLS certificate keys and add them to Kubernetes secrets:
```console
openssl req -x509 -newkey rsa:2048 -keyout tls.key -out tls.crt -nodes -subj '/CN=echo-server'
```

Adding TLS files to Kubernetes secrets

```console
kubectl create secret tls echo-tls --cert=tls.crt --key=tls.key
```

### Kubernetes sidecar deployment
The following configuration defines the main application containers, “nodejs-hello” and nginx container “nginx”. Both containers run in the same pod and share pod resources and serve as an example implementation of the sidecar pattern. One thing you want to modify is hostname, you can us a non existing hostname appname.example.com for this example.

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: sidecar-ex
  labels:
    app: sidecar-ex
    proxy: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sidecar-ex
  template:
    metadata:
      labels:
        app: sidecar-ex
    spec:
      containers:
      - name: echo
        image: evillgenius/echo-server:vb1
        ports:
        - containerPort: 8080
      - name: nginx
        image: evillgenius/tls-sidecar:vb1
        volumeMounts:
          - name: ssl-keys
            readOnly: true
            mountPath: "/etc/nginx/ssl"          
        ports:
        - containerPort: 443
      volumes:
      - name: ssl-keys
        secret:
          secretName: echo-tls
```


Save this file to sidecar-ex.yaml and create deployment Kubernetes object:

```console
kubectl create -f sidecar-ex.yaml
```

Wait for pods to be Ready:
```console
kubectl get pods

NAME                            READY     STATUS    RESTARTS   AGE
sidecar-ex-686bbff8d7-42mcn   2/2       Running   0          1m
```
## Testing
For testing, setup a port forwarding rule. This will map a loca port 8043 to the ssl sidecar on port 443:

```console
kubectl port-forward <pod> 8043:443 &
```

First lets validate that application does not respond on http and then it does respond on https requests


### Using http
```console
curl -k http://127.0.0.1:8043/lorem
```

```output
<html>
<head><title>400 The plain HTTP request was sent to HTTPS port</title></head>
<body bgcolor="white">
<center><h1>400 Bad Request</h1></center>
<center>The plain HTTP request was sent to HTTPS port</center>
<hr><center>nginx/1.11.13</center>
</body>
</html>
```
### Using https
```console
curl -k https://127.0.0.1:8043/lorme 
```

```output
Lorem ipsum dolor sit amet, consectetur adipiscing elit. Praesent a scelerisque nisl. Sed placerat finibus ante, non iaculis dui. Etiam viverra, ex sit amet scelerisque lacinia, est nulla egestas lectus, sit amet ullamcorper nisi nisl vitae orci. Fusce et dui facilisis, luctus mauris sed, consequat arcu. Duis porttitor libero id neque dapibus, eget aliquet turpis accumsan. Praesent lectus mauris, tempor eu gravida at, hendrerit nec massa. Vestibulum sed luctus est. Nunc rutrum risus at nisl dictum volutpat. Pellentesque auctor massa tortor, consequat tincidunt tortor bibendum a. Nullam mattis eros eu risus volutpat, id euismod elit facilisis. Vestibulum quis eros a magna accumsan scelerisque vel vel metus. Curabitur at magna cursus, vulputate velit eu, sodales dolor. Duis cursus, tortor in vehicula tempor, quam nisi dictum arcu, eu congue mauris felis quis quam. Vivamus vitae risus venenatis, tincidunt eros ac, posuere purus.

Nunc placerat feugiat hendrerit. Proin ipsum ex, tincidunt sed ante et, pretium ullamcorper mauris. Pellentesque vel dolor sed dolor euismod vestibulum. Cras facilisis non lacus nec tincidunt. Duis pharetra lacinia risus, nec sollicitudin odio. Sed et iaculis sapien. Interdum et malesuada fames ac ante ipsum primis in faucibus. Sed ultricies rhoncus nunc, a consequat neque pulvinar id. Phasellus in tincidunt tellus, in blandit sem. Integer interdum vehicula lacus at tincidunt. Donec feugiat ultricies risus, et pellentesque orci porta mattis. Morbi ut nisl ut metus ornare egestas. Donec eget orci vitae justo hendrerit faucibus. Morbi et feugiat velit. Integer at iaculis tellus. Aenean dolor lacus, blandit a tempus eu, malesuada ac massa.

Sed suscipit ac mi vitae tincidunt. Vestibulum auctor ...
```

Find and kill the portforward kubectl processes.
```console
ps | grep kubectl
kill <pids of kubectl processes>
```

Great! We have got expected output through https connection.


