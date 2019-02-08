# Using Advanced Deployment API Features

## Update Strategy
Other than providing seamless replication and declarative system states, using deployments gives you the ability to update your application with zero downtime and track multiple versions of your deployment environment.
When a deployment's configuration changes—for example by updating the image used by its pods—Kubernetes detects this change and executes steps to reconcile the system state with the configuration change. The mechanism by which Kubernetes executes these updates is determined by the deployment's strategy.type field.

### Recreate

With the Recreate strategy, all running pods in the deployment are killed, and new pods are created in their place. This does not guarantee zero- downtime, but can be useful in case you are not able to have multiple versions of your application running simultaneously.

### RollingUpdate

The preferred and more commonly used strategy is RollingUpdate. This gracefully updates pods one at a time to prevent your application from going down. The strategy gradually brings pods with the new configuration online, while killing old pods as the new configuration scales up.

When updating your deployment with RollingUpdate, there are two useful fields you can configure:

1. maxUnavailable effectively determines the minimum number of pods you want running in your deployment as it updates. For example, if we have a deployment currently running ten pods and a maxUnavailable value of 4. When an update is triggered, Kubernetes will immediately kill four pods from the old configuration, bringing our total to six. Kubernetes then starts to bring up the new pods, and kills old pods as they come alive. Eventually the deployment will have ten replicas of the new pod, but at no point during the update were there fewer than six pods available.

2. maxSurge determines the maximum number of pods you want running in your deployment as it updates. In the previous example, if we specified a maxSurge of 3, Kubernetes could immediately create three copies of the new pod, bringing the total to 13, and then begin killing off the old versions.

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

A PDB specifies the number of replicas that an application can tolerate having, relative to how many it is intended to have. For example, a Deployment which has a replicas: 5 is supposed to have 5 pods at any given time. If its PDB allows for there to be 4 at a time, then the Eviction API will allow voluntary disruption of one, but not two pods, at a time.

**PDBs cannot prevent involuntary disruptions from occurring, but they do count against the budget.**

Pods which are deleted or unavailable due to a rolling upgrade to an application do count against the disruption budget, but controllers (like deployment and stateful-set) are not limited by PDBs when doing rolling upgrades – the handling of failures during application updates is configured in the controller.

When a pod is evicted using the eviction API, it is gracefully terminated

### Lab 2.6 - Create a deployment with replicas, an update strategy and assign a Pod disruption budget

First create a very basic deployment that will deploy an nginx container with nginx 1.7.9

```yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
