# Exercise 1.3 - Deploy an Ambassador Container

We need to connect to an external resource, however we want to be able to
dynamically switch our external connection without changing the application
container.

User request => application => ambassador => external service

## Main application
For our example main application use a Nodejs web service that connects to the external resource via the ambassador, which is essentially a proxy. 
## Ambassador container 
This container is a simple NodeJS web client that returns a random external API endpoint when called.
## Deployment
### Kubernetes ambassador deployment
The following configuration defines the main application container “app-container” and the ambassador container “ambassador-container”. Both containers run in the same pod and share pod resources including localhost network. 

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: ambassador
  labels:
    app: ambassador-ex
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ambassador-ex
  template:
    metadata:
      labels:
        app: ambassador-ex
    spec:
      containers:
      - name: app-container
        image: evillgenius/ambassador-example-app:1.0.0
        ports:
        - containerPort: 3000
      - name: ambassador-container
        image: evillgenius/ambassador-example-ambassador:1.0.0          
        ports:
        - containerPort: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: ambassador-svc
spec:
  # Select which pods can be used to handle traffic to this service
  # based on their labels
  selector:
    app: ambassador-ex

  # Expose the service on a static port on each node
  # so that we can access the service from outside the cluster
  type: LoadBalancer

  # Ports exposed by the service.
  ports:
    # `port` is the port for the service.
    # `targetPort` is the port for the pods,
    #  it has to match what's defined in the pod YAML.
    - port: 80
      targetPort: 3000
```


Save this file to ambassador-ex.yaml and create deployment Kubernetes object:

```console
kubectl create -f ambassador-ex.yaml
```

Wait for pods to be running:
```console
kubectl get pods
```

```output
NAME                            READY     STATUS    RESTARTS   AGE
ambassador-686bbff8d7-42mcn   2/2       Running   0          1m
```
## Testing
The manifest included a service of type LoadBalancer which exposes the app-container port 3000 on port 80 of an Azure Load Balancer. Determine the external IP address of the Load Balancer by:

```console
kubectl get svc
```

```output
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
ambassador-svc   LoadBalancer   172.23.186.163   40.117.238.238   80:30429/TCP   5m
kubernetes       ClusterIP      172.23.0.1       <none>           443/TCP        17d
```
In the example above the output shows the ambassador-svc EXTERNAL-IP is 40.117.238.238 and is listening on port 80. 
>**Note:** Your External IP address will be different so be sure to use your resulting IP address for the test

Now open your browser or from the command line you can curl the public IP

```console
curl 40.117.238.238
```

Run the command a few times and the output will randomly change

**EXAMPLE OUTPUT**

```output
eddie@Azure:~$ curl 40.117.238.238
{"page":2,"per_page":3,"total":12,"total_pages":4,"data":[{"id":4,"first_name":"Eve",>"last_name":"Holt","avatar":"https://s3.amazonaws.com/uifaces/faces/twitter/marcoramires/128.jpg"},{"id":5,>"first_name":"Charles","last_name":"Morris","avatar":"https://s3.amazonaws.com/uifaces/faces/twitter/stephenmoon/128.jpg"},{"id":6,>"first_name":"Tracey","last_name":"Ramos","avatar":"https://s3.amazonaws.com/uifaces/faces/twitter/bigmancho/128.jpg"}]}
```
```output
eddie@Azure:~$ curl 40.117.238.238
{
  "userId": 1,
  "id": 1,
  "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
  "body": "quia et suscipit\nsuscipit recusandae consequuntur expedita et cum\nreprehenderit molestiae ut ut quas totam\nnostrum rerum est autem sunt rem eveniet architecto"
}
```
Great! We have got expected output through the Ambassador proxy.
