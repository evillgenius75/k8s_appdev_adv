# Kubernetes Metrics Architecture

## Metrics Server

Starting from Kubernetes 1.8, resource usage metrics, such as container CPU and memory usage, are available in Kubernetes through the Metrics API. These metrics can be either accessed directly by user, for example by using kubectl top command, or used by a controller in the cluster, e.g. Horizontal Pod Autoscaler, to make decisions.

Through the Metrics API you can get the amount of resources currently used by a given node or a given pod. This API doesn’t store the metric values, so it’s not possible for example to get the amount of resources used by a given node 10 minutes ago.

The API is no different from any other API:

It is discoverable through the same endpoint as the other Kubernetes APIs under /apis/metrics.k8s.io/ path
it offers the same security, scalability and reliability guarantees.

Metrics Server is a cluster-wide aggregator of resource usage data. Starting from Kubernetes 1.8 it’s deployed by default in clusters as a Deployment object. 

Metric server collects metrics from the Summary API, exposed by Kubelet on each node.

The Metrics Server is registered to the main API server through Kubernetes aggregator.

## Horizontal Pod Autoscaler

The Horizontal Pod Autoscaler automatically scales the number of pods in a replication controller, deployment or replica set based on observed CPU utilization (or, with custom metrics support, on some other application-provided metrics). Note that Horizontal Pod Autoscaling does not apply to objects that can’t be scaled, for example, DaemonSets.

The Horizontal Pod Autoscaler is implemented as a Kubernetes API resource and a controller. The resource determines the behavior of the controller. The controller periodically adjusts the number of replicas in a replication controller or deployment to match the observed average CPU utilization to the target specified by user.

## Lab 3.1 Implement an autoscale based on an external metric

Validate your metrics server is accessable by running:
```console
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .
```
Create a service account for tiller and give it Kubernetes RBAC to do the thing tiller needs to do. First create a file called tillerrbac.yaml and populate it with the below yaml:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
Now create the above objects using kubectl:

```console
kubectl create -f tillerrbac.yaml
```
Last, we need to intitialize helm with our aks cluster:

```console
helm init --service-account tiller
```

Clone the external metrics adapter project and move into the servicebus example directory:

```console
git clone github.com/Azure/azure-k8s-metrics-adapter
cd azure-k8s-metrics-adapter/samples/servicebus-queue/
```

## Setup Service Bus
Create a service bus in azure:

```console
export SERVICEBUS_NS=sb-external-ns-<your-initials>
```
```console
az group create -n sb-external-example -l eastus
```
```console
az servicebus namespace create -n $SERVICEBUS_NS -g sb-external-example
```
```cosnole
az servicebus queue create -n externalq --namespace-name $SERVICEBUS_NS -g sb-external-example
```

Create an auth rules for queue:

```console
az servicebus queue authorization-rule create --resource-group sb-external-example --namespace-name $SERVICEBUS_NS --queue-name externalq  --name demorule --rights Listen Manage Send

#save for connection string for later
export SERVICEBUS_CONNECTION_STRING="$(az servicebus queue authorization-rule keys list --resource-group sb-external-example --namespace-name $SERVICEBUS_NS --name demorule  --queue-name externalq -o json | jq -r .primaryConnectionString)"
```

> **Note:** this gives full access to the queue for ease of use of demo.  You should create more fine grained control for each component of your app.  For example the consumer app should only have `Listen` rights and the producer app should only have `Send` rights.

## Setup AKS Cluster

### Enable Access to Azure Resources

Run the scripts provided to [configure a Service Principal](https://github.com/Azure/azure-k8s-metrics-adapter/blob/master/README.md#using-azure-ad-application-id-and-secret) with the following environment variables for giving the access to the Service Bus Namespace Insights provider.

### Start the producer
Make sure you have cloned this repository and are in the folder `samples/servicebus-queue` for remainder of this walkthrough.

download the producer app to produce some messages in the queue
```console
wget -O sb-producer https://ejvlab110533.blob.core.windows.net/ejvlab/producer
```

Run the producer to create a few queue items, then hit `ctl-c` after a few message have been sent to stop it:

```console
chmod +x sb-producer
./sb-producer 500
```

Check the queue has values:

```console
az servicebus queue show --resource-group sb-external-example --namespace-name $SERVICEBUS_NS --name externalq -o json | jq .messageCount
```

### Configure Secret for consumer pod
Create a secret with the connection string (from [previous step](#setup-service-bus)) for the service bus:

```console
kubectl create secret generic servicebuskey --from-literal=sb-connection-string=$SERVICEBUS_CONNECTION_STRING
```

### Deploy Consumer 
Deploy the consumer:

```console
kubectl apply -f deploy/consumer-deployment.yaml
```

Check that the consumer was able to receive messages:

```console
kubectl logs -l app=consumer
```

```output
connecting to queue:  externalq
setting up listener
listening...
received message:  the answer is 42
number message left:  6
received message:  the answer is 42
number message left:  5
received message:  the answer is 42
number message left:  4
received message:  the answer is 42
number message left:  3
received message:  the answer is 42
number message left:  2
received message:  the answer is 42
```

## Set up Azure Metrics Adapter

### Deploy the adapter

Deploy the adapter:

Create a Service Principal
```console
az ad sp create-for-rbac 
```

Use the output from the result in the fields of the helm chart below:
```console
helm install --name sample-release ../../charts/azure-k8s-metrics-adapter --namespace custom-metrics --set azureAuthentication.method=clientSecret --set azureAuthentication.tenantID=<your tenantid> --set azureAuthentication.clientID=<your clientID> --set azureAuthentication.clientSecret=<your clientSecret> --set azureAuthentication.createSecret=true`
```


Check you can hit the external metric endpoint.  The resources will be empty as it [is not implemented yet](https://github.com/Azure/azure-k8s-metrics-adapter/issues/3) but you should receive a result.

```console
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" | jq .
```
```output
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "external.metrics.k8s.io/v1beta1",
  "resources": []
}
```

### Configure Metric Adapter with metrics
The metric adapter deploys a CRD called ExternalMetric which you can use to configure metrics.  To deploy these metric we need to update the Service Bus namespace in the configuration then deploy it:

```console
sed -i 's|sb-external-ns|'${SERVICEBUS_NS}'|g' deploy/externalmetric.yaml
```
```console
kubectl apply -f deploy/externalmetric.yaml
```

> **Note:** the ExternalMetric configuration is deployed per namespace.

You can list of the configured external metrics via:

```console
kubectl get aem
```

### Deploy the HPA
Deploy the HPA:

```console
kubectl apply -f deploy/hpa.yaml
```

> **Note:** the `external.metricName` defined on the HPA must match the `metadata.name` on the ExternalMetric declaration, in this case `queuemessages`

After a few seconds, validate that the HPA is configured.  If the `targets` shows `<unknown>` wait longer and try again.

```console
kubectl get hpa consumer-scaler
```

You can also check the queue value returns manually by running:

```console
kubectl  get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/default/queuemessages" | jq .
```

## Scale!

Put some load on the queue. Note this will add 20,000 message then exit.

```console
./sb-producer 0 > /dev/null &
```

Now check your queue is loaded:

```console
az servicebus queue show --resource-group sb-external-example --namespace-name $SERVICEBUS_NS --name externalq -o json | jq .messageCount
```
```output
// should have a good 19,000 or more
19,858
```

Now watch your HPA pick up the queue count and scal the replicas.  This will take 2 or 3 minutes due to the fact that the HPA check happens every 30 seconds and must have a value over target value.  It also takes a few seconds for the metrics to be reported to the Azure endpoint that is queried.  It will then scale back down after a few minutes as well.

```console
kubectl get hpa consumer-scaler -w
```

```output
NAME              REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE  
consumer-scaler   Deployment/consumer   0/30      1         10        1          1h   
consumer-scaler   Deployment/consumer   27278/30   1         10        1         1h   
consumer-scaler   Deployment/consumer   26988/30   1         10        4         1h   
consumer-scaler   Deployment/consumer   26988/30   1         10        4         1h           consumer-scaler   Deployment/consumer   26702/30   1         10        4         1h           
consumer-scaler   Deployment/consumer   26702/30   1         10        4         1h           
consumer-scaler   Deployment/consumer   25808/30   1         10        4         1h           
consumer-scaler   Deployment/consumer   25808/30   1         10        4         1h           consumer-scaler   Deployment/consumer   24784/30   1         10        8         1h           consumer-scaler   Deployment/consumer   24784/30   1         10        8         1h          
consumer-scaler   Deployment/consumer   23775/30   1         10        8         1h           
consumer-scaler   Deployment/consumer   22065/30   1         10        8         1h           
consumer-scaler   Deployment/consumer   22065/30   1         10        8         1h           
consumer-scaler   Deployment/consumer   20059/30   1         10        8         1h           
consumer-scaler   Deployment/consumer   20059/30   1         10        10        1h
```

Once it is scaled up you can check the deployment:

```console
kubectl get deployment consumer
```

```output
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
consumer   10         10         10            10           23m
```

And check out the logs for the consumers:

```console
kubectl logs -l app=consumer --tail 100
```

## Clean up
Once the queue is empty (will happen pretty quickly after scaled up to 10) you should see your deployment scale back down.

Once you are done with this experiment you can delete kubernetes deployments:

```console
kubectl delete -f deploy/hpa.yaml
kubectl delete -f deploy/consumer-deployment.yaml
helm delete --purge sample-release
```
