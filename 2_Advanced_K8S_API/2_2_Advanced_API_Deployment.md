# Using Advanced Deployment API Features

## Deployment API

In Kubernetes, Deployment is the recommended way to deploy applications into Kubernetes and is now the main Object with the older `ReplicationControllers` being deprecated. Below are some of the key benefits of the Kubernetes `Deployment`object.

  * Deploys a ReplicaSet that manages Pod replicas.
  * Updates pods (PodTemplateSpec).
  * Rollback to older Deployment versions.
  * Scale Deployment up or down.
  * Pause and resume the Deployment.
  * Use the status of the Deployment to determine state of replicas.
  * Clean up older ReplicaSets that you don’t need anymore.
  * Canary Deployment and Blue/Green Deployment capabilities.

### Lab 2.6 - Create a deployment with replicas.

First create a very basic deployment that will deploy an nginx containers with nginx version 1.7

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7
        ports:
        - containerPort: 80
```
Save this file to base-deploy.yaml and create pod Kubernetes object:

```console
kubectl apply -f base-deploy.yaml --record=true
```

Wait for pods to be Ready:
```console
kubectl get pods -w 

NAME                                READY     STATUS    RESTARTS   AGE
nginx-deploy-6bbdfbd484-7x8kp       1/1       Running   0          45s
```

`ctrl+c to` exit the wait state

## Update Strategy
Other than providing seamless replication and declarative system states, using deployments gives you the ability to update your application with zero downtime and track multiple versions of your deployment environment.
When a deployment's configuration changes—for example by updating the image used by its pods—Kubernetes detects this change and executes steps to reconcile the system state with the configuration change. The mechanism by which Kubernetes executes these updates is determined by the deployment's `strategy.type` field.

### Recreate

With the Recreate strategy, all running pods in the deployment are killed, and new pods are created in their place. This does not guarantee zero- downtime, but can be useful in case you are not able to have multiple versions of your application running simultaneously.

### Lab 2.7 - Update a deployment with Recreate Strategy and updte container.

Let's edit the deployment so that the update strategy is recreate.

```console
kubectl edit deploy nginx-deploy
```
This will open a VI editor in console. Replace the lines:
```yaml
strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
```
with the lines:
```yaml
strategy:
    type: Recreate
```
save the file using vi commands and now inspect the deployment to verify the change was made

```console
kubetctl get deploy nginx-deploy -o yaml
```

Now scale the deployment to 6 replicas

```console
kubectl scale deploy nginx-deploy --replicas=6 --record=true

kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deploy-6bbdfbd484-8dl98       1/1       Running   0          6s
nginx-deploy-6bbdfbd484-brxqw       1/1       Running   0          4m
nginx-deploy-6bbdfbd484-bzbjc       1/1       Running   0          6s
nginx-deploy-6bbdfbd484-m7mpv       1/1       Running   0          6s
nginx-deploy-6bbdfbd484-s84m9       1/1       Running   0          6s
nginx-deploy-6bbdfbd484-spmq4       1/1       Running   0          6s
```

Now we will patch the deployment to have a new version of nginx from 1.7 to 1.9

```console
kubectl set image deploy nginx-deploy nginx=nginx:1.9 --record=true
kubectl get pods -w
```

What you should see is all of the original pods be terminated and then deleted, and rather quickly 6 new pods pods being created with the new version of nginx.

### RollingUpdate

The preferred and more commonly used strategy is `RollingUpdate`. This gracefully updates pods one at a time to prevent your application from going down. The strategy gradually brings pods with the new configuration online, while killing old pods as the new configuration scales up.

When updating your deployment with RollingUpdate, there are two useful fields you can configure:

1. `maxUnavailable` effectively determines the minimum number of pods you want running in your deployment as it updates. For example, if we have a deployment currently running ten pods and a maxUnavailable value of 4. When an update is triggered, Kubernetes will immediately kill four pods from the old configuration, bringing our total to six. Kubernetes then starts to bring up the new pods, and kills old pods as they come alive. Eventually the deployment will have ten replicas of the new pod, but at no point during the update were there fewer than six pods available.

2. `maxSurge` determines the maximum number of pods you want running in your deployment as it updates. In the previous example, if we specified a maxSurge of 3, Kubernetes could immediately create three copies of the new pod, bringing the total to 13, and then begin killing off the old versions.

### Lab 2.8 Change the Strategy to RollingUpdate leaving at least 4 pods of the old service available and no more than 10 pods at any given time

Let's edit the deployment so that the update strategy is recreate.

```console
kubectl edit deploy nginx-deploy
```
This will open a VI editor in console. Replace the lines:

```yaml
strategy:
    type: Recreate
```

with the lines:

```yaml
strategy:
    rollingUpdate:
      maxSurge: 4
      maxUnavailable: 2
    type: RollingUpdate
```

save the file using vi commands and now inspect the deployment to verify the change was made

```console
kubetctl get deploy nginx-deploy -o yaml
```

Now we will patch the deployment to have a new version of nginx from 1.9 to 1.10

```console
kubectl set image deploy nginx-deploy nginx=nginx:1.10 --record=true
kubectl get pods -w
```

You will see that 4 new pods will be created and 2 old pods will be in tern=minating state and you can watch the rollout of the deployment happen.

##Rollbacks
The rollback process in v1 of the Deployment API is now a manual process through the `kubectl rollout` command set. 

### Lab 2.9 Roll back to nginx version 1.7

Lets inspect the rollout history and rollback the deployment to use version 1.7 of nginx

```console
kubectl rollout history deploy nginx-deploy

deployments "nginx-deploy"
REVISION  CHANGE-CAUSE
1         kubectl scale deploy nginx-deploy --replicas=6 --record=true
2         kubectl set image deploy nginx-deploy nginx=nginx:1.9 --record=true
3         kubectl set image deploy nginx-deploy nginx=nginx:1.10 --record=true
```
Now review the revision 1 to verify it will revert the image back to 1.7 of nginx

```console
kubectl rollout history deploy nginx-deploy --revision=1 --record=true

kubectl describe deploy nginx-deploy
Name:                   nginx-deploy
Namespace:              default
CreationTimestamp:      Sun, 10 Feb 2019 21:10:23 +0000
Labels:                 app=nginx
...
Containers:
   nginx:
    Image:        nginx:1.7
    Port:         80/TCP
...
```

Now the pod is running nginx version 1.7 again.

## Pod Disruption Budgets

Pods do not disappear until someone (a person or a controller) destroys them, or there is an unavoidable hardware or system software error.

We call these unavoidable cases **involuntary** disruptions to an application. Examples are:

  * a hardware failure of the physical machine backing the node
  * cluster administrator deletes VM (instance) by mistake
  * cloud provider or hypervisor failure makes VM disappear
  * a kernel panic
  * the node disappears from the cluster due to cluster network partition
  * eviction of a pod due to the node being out-of-resources.

We call other cases **voluntary** disruptions. These include both actions initiated by the application owner and those initiated by a Cluster Administrator. Typical application owner actions include:

  * deleting the deployment or other controller that manages the pod
  * updating a deployment’s pod template causing a restart
  * directly deleting a pod (e.g. by accident)

Cluster Administrator actions include:

  * Draining a node for repair or upgrade.
  * Draining a node from a cluster to scale the cluster down (learn about Cluster Autoscaling ).
  * Removing a pod from a node to permit something else to fit on that node.

These actions might be taken directly by the cluster administrator, or by automation run by the cluster administrator, or by your cluster hosting provider.

An Application Owner can create a PodDisruptionBudget object (PDB) for each application. A PDB limits the number of pods of a replicated application that are down simultaneously from voluntary disruptions. 

Cluster managers and hosting providers should use tools which respect Pod Disruption Budgets by calling the Eviction API instead of directly deleting pods. Examples are the `kubectl drain` command and the `az aks upgrade` command on AKS.

A PDB specifies the number of replicas that an application can tolerate having, relative to how many it is intended to have. For example, a Deployment which has a `replicas: 5` is supposed to have 5 pods at any given time. If its PDB allows for there to be 4 at a time, then the Eviction API will allow voluntary disruption of one, but not two pods, at a time.

**PDBs cannot prevent involuntary disruptions from occurring, but they do count against the budget.**

Pods which are deleted or unavailable due to a rolling upgrade to an application do count against the disruption budget, but controllers (like deployment and stateful-set) are not limited by PDBs when doing rolling upgrades – the handling of failures during application updates is configured in the controller.

When a pod is evicted using the eviction API, it is gracefully terminated

### Lab 2.9 Create a PDB for nginx-deploy that allows 5 pods at a time then drain the node with the most replicas

First determine which node has the most nginx pods

```console
kubectl get pods -o wide
NAME                            READY     STATUS    RESTARTS   AGE       IP            NODE
nginx-deploy-6bbdfbd484-7d4v8   1/1       Running   0          3m        10.244.1.19   aks-nodepool1-76410264-2
nginx-deploy-6bbdfbd484-8chs4   1/1       Running   0          3m        10.244.2.20   aks-nodepool1-76410264-0
nginx-deploy-6bbdfbd484-9s9z7   1/1       Running   0          3m        10.244.2.19   aks-nodepool1-76410264-0
nginx-deploy-6bbdfbd484-9xk97   1/1       Running   0          3m        10.244.0.14   aks-nodepool1-76410264-1
nginx-deploy-6bbdfbd484-p69ht   1/1       Running   0          3m        10.244.1.20   aks-nodepool1-76410264-2
nginx-deploy-6bbdfbd484-wt6g6   1/1       Running   0          3m        10.244.2.18   aks-nodepool1-76410264-0
```
In the sample output above node0 has 3 of the 6 replicas. This will be the node that will be drained

Now create the Pod Disruption Budget for the Deployment

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 5
  selector:
    matchLabels:
      app: nginx
```

Save the files a nginx-pdb.yaml and apply it to the cluster

```console
kubectl apply -f nginx-pdb.yaml
```

Verify that the disruption budget reflects the new state of the deployment

```console
kubectl get pdb nginx-pdb -o yaml 
```

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
...
spec:
  minAvailable: 5
  selector:
    matchLabels:
      app: nginx
status:
  currentHealthy: 6
  desiredHealthy: 5
  disruptedPods: null
  disruptionsAllowed: 1
  expectedPods: 6
  observedGeneration: 1
```

Now drain the node you selected as the one with the most replicas
>**Note:** You may want to open another browser window with http://shell.azure.com and run the following commands in two seperate shells. Run the watch command first and then drain the node you selected

```console
kubectl drain node aks-nodepool1-76410264-0
```

  Use the watch command to see the updates happen in real time and you will notice that only a sinle container is evicted at a time to preserve the PDB created.

```console
watch -d -n 1 kubectl get pods -o wide
```
