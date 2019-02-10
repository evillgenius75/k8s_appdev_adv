# Exercise 1.4 - Deploy an Adapter Container

## Main application
It defines a main application container which writes the current date and system usage information to a log file every five seconds. 
## Adapter container 
The adapter container reads what the application has written andreformats it into a structure that a hypothetical monitoring service requires.

## Deployment
### Kubernetes adapter deployment
This application writes system usage information `top` to a status file every five seconds. This sidecar container takes the output format of the application(the current date and system usage information), simplifies and reformats it for the monitoring service to come and collect. In this example, our monitoring service requires status files to have the date, then memory usage, then CPU percentage each on a new line. Our adapter container will inspect the contents of the app's top file, reformat it, and write the correctly formatted output to the status file.


```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: adapter
  labels:
    app: adapter-ex
spec:
  replicas: 1
  selector:
    matchLabels:
      app: adapter-ex
  template:
    metadata:
      labels:
        app: adapter-ex
    spec:
  # Create a volume called 'shared-logs' that the
  # app and adapter share.
      volumes:
      - name: shared-logs 
        emptyDir: {}
      containers:
      # Main application container
      - name: app-container
        image: alpine
        command: ["/bin/sh"]
        args: ["-c", "while true; do date > /var/log/top.txt && top -n 1 -b >> /var/log/top.txt; sleep 5;done"]
  # Mount the pod's shared log file into the app 
  # container. The app writes logs here.
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log

# Adapter container
      - name: adapter-container
        image: alpine
        command: ["/bin/sh"]

  # A long command doing a simple thing: read the `top.txt` file that the
  # application wrote to and adapt it to fit the status file format.
  # Get the date from the first line, write to `status.txt` output file.
  # Get the first memory usage number, write to `status.txt`.
  # Get the first CPU usage percentage, write to `status.txt`.

        args: ["-c", "while true; do (cat /var/log/top.txt | head -1 > /var/log/status.txt) && (cat /var/log/top.txt | head -2 | tail -1 | grep
-o -E '\\d+\\w' | head -1 >> /var/log/status.txt) && (cat /var/log/top.txt | head -3 | tail -1 | grep
-o -E '\\d+%' | head -1 >> /var/log/status.txt); sleep 5; done"]


  # Mount the pod's shared log file into the adapter container.
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log
```


Save this file to adapter-ex.yaml and create deployment Kubernetes object:

```console
kubectl create -f adapter-ex.yaml
```

Wait for pods to be running:
```console
kubectl get pods

NAME                            READY     STATUS    RESTARTS   AGE
adapter-5469c8c6f9-cdcpp      2/2     Running   0          5s
```

>**NOTE:** The Pod name in the output above is an example. Please use your exact pod name in the steps below

## Testing
Once the pod is running connect to the application pod:

```console
kubectl exec <your-pod-name> -c app-container -it sh
``` 

Take a look at what the application is writing:

```console
cat /var/log/top.txt
```   
Take a look at what the adapter has reformatted it to:

```console
cat /var/log/status.txt
```

Now the logging system can receive the log in the format it expects the data to be in and the original application did not have to be modified in any way.