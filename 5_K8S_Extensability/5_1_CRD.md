# Extending Kubernetes with Custom Resources

Let’s assume you want to build a clustered application or a software as a service offering. Before you start to write one single line of application code, you must address an overflowing array of architectural issues, including security, multitenancy, API gateways, CLI, configuration management, and logging.

What if you could just leverage all that infrastructure from Kubernetes, save yourself few man years in development, and focus on implementing your unique service?

The latest Kubernetes adds an important feature called CustomResourceDefinitions (CRD), which enables plugging in your own managed object and application as if it were a native Kubernetes component. This way, you can leverage Kubernetes CLI, API services, security and cluster management frameworks without modifying Kubernetes or knowing its internals.

Since this is our last lab let's have some fun! Let's make a Game Server using Kubernetes with Project Agones!

## Lab 5.1 - "Do You Want To Play A Game?"

### Allowing UDP traffic
For Agones to work correctly, we need to allow UDP traffic to pass through to our AKS cluster. To achieve this, we must update the NSG (Network Security Group) with the proper rule. A simple way to do that is:

#### Update Network Security
Get the underlying resource group for the AKS cluster then get the Network Security Group that needs updating:

```console
RESOURCE_GROUP=$(az group list --output json --query "[?starts_with(name, 'MC_k8s_appdev_adv')].name" | jq .[0] -r)
NSG=$(az network nsg list -g $RESOURCE_GROUP --output json --query "[].name" | jq .[0] -r)
```

Create a new Rule with UDP as the protocol and 7000-8000 as the Destination Port Ranges. 

```console
az network nsg rule create --name allow-game-inbound \
--nsg-name $NSG \
--resource-group $RESOURCE_GROUP \
--priority 100 \
--direction  Inbound \
--access Allow \
--protocol UDP \
--destination-port-ranges 7000-8000
```

### Creating and assigning Public IPs to Nodes
Nodes in AKS don’t get a Public IP by default. One way is to manually assign a Public IP to each node by finding the Resource Group where the AKS resources are installed on the portal (it should have a name like MC_resourceGroupName_AKSName_regionName), then adding a Public IP to each VM (this can also be done through the CLI). 

Alternatively, a DaemonSet can be deployed to the cluster that will automatically assign a PublicIP to each node of your cluster. You will use this approach.

```console
kubectl create -n kube-system -f https://raw.githubusercontent.com/dgkanatsios/AksNodePublicIPController/master/deploy.yaml
```

### Enabling creation of RBAC resources
To install Agones, a service account needs permission to create some special RBAC resource types.

```console
kubectl create clusterrolebinding cluster-admin-binding \
--clusterrole=cluster-admin \
--serviceaccount=kube-system:default
```

### Installing Agones
We can install Agones to the cluster using the install.yaml file on GitHub. We will pass the URI to the file. (You can also find the install.yaml in the latest agones-install zip from the releases archive if you prefer that approach.)

```console
kubectl create namespace agones-system
kubectl apply -f https://github.com/GoogleCloudPlatform/agones/raw/release-0.7.0/install/yaml/install.yaml
```

#### Confirm that Agones started successfully
To confirm Agones is up and running, run the following command:

```console
kubectl describe --namespace agones-system pods
```

It should describe the single pod created in the agones-system namespace, with no error messages or status. The Conditions section should look like this:

```output
...
Conditions:
  Type           Status
  Initialized    True
  Ready          True
  PodScheduled   True
```

That’s it! This creates the Custom Resource Definitions that power Agones and allows us to define resources of type GameServer.

For the purpose of this guide we’re going to use the simple-udp example as the GameServer container. It's a very simple UDP server written in Go. Don’t hesitate to look at the code of this example for more information.

### Create a GameServer
Let’s create a GameServer using the following command :

```console
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/agones/release-0.7.0/examples/simple-udp/gameserver.yaml
```

You should see a successful output similar to this:

```console
gameserver.stable.agones.dev/simple-udp created
```

This has created a GameServer record inside Kubernetes, which has also created a backing Pod to run our simple udp game server code in. If you want to see all your running GameServers you can run:

```console
kubectl get gameservers
```

It should look something like this:

```console
NAME         AGE
simple-udp   5m
```

You can also see the Pod that got created by running `kubectl get pods`, the Pod will be prefixed by simple-udp.

```console
NAME                                     READY     STATUS    RESTARTS   AGE
simple-udp-vwxpt                         2/2       Running   0          5m
```

As you can see above it says `READY: 2/2` this means there are two containers running in this Pod, this is because Agones injected the SDK sidecar for readiness and health checking of your Game Server.


### Fetch the GameServer Status
Let’s wait for the GameServer state to become Ready:

```console
watch kubectl describe gameserver
```
```output
Name:         simple-udp-jq8kd-q8dzg
Namespace:    default
Labels:       stable.agones.dev/gameserverset=simple-udp-jq8kd
Annotations:  <none>
API Version:  stable.agones.dev/v1alpha1
Kind:         GameServer
...
Status:
  Address:    192.168.99.100
  Node Name:  agones
  Ports:
    Name:  default
    Port:  7614
  State:   Ready
Events:
  Type    Reason          Age   From                   Message
  ----    ------          ----  ----                   -------
  Normal  PortAllocation  23s   gameserver-controller  Port allocated
  Normal  Creating        23s   gameserver-controller  Pod simple-udp-jq8kd-q8dzg-9kww8 created
  Normal  Starting        23s   gameserver-controller  Synced
  Normal  Ready           20s   gameserver-controller  Address and Port populated
```

If you look towards the bottom, you can see there is a `Status > State` value. We are waiting for it to move to Ready, which means that the game server is ready to accept connections.

You might also be interested to see the Events section, which outlines when various lifecycle events of the GameSever occur. We can also see when the GameServer is ready on the event stream as well - at which time the Status > Address and Status > Port have also been populated, letting us know what IP and port our client can now connect to!

### Let’s retrieve the IP address and the allocated port of your Game Server

```console
kubectl get gs -o=custom-columns=NAME:.metadata.name,STATUS:.status.state,IP:.status.address,PORT:.status.ports
```

This will output your Game Server IP address and ports, eg:

```console
NAME         STATUS    IP               PORT
simple-udp   Ready     192.168.99.100   [map[name:default port:7614]]
```

### Connect to the GameServer

You can now communicate with the Game Server:

> NOTE: if you do not have netcat installed (i.e. you get a response of nc: command not found), you can install netcat by running sudo apt install netcat. 
NetCat is not installed in the Azure Cloudshell so we will need to temporarily run a utility pod that will allow us to do the call

```console
 kubectl run busybox -i --tty --image=busybox --restart=Never --rm -- sh
 ```
 ```output
 If you don't see a command prompt, try pressing enter.
/ #
```

 Now use the `nc` command 
```console
nc -u {IP} {PORT}
```
Now type a message the game will `ACK` back to you:
```console
Hello World !
```
```output
ACK: Hello World !
```
If the message returned successfully you can finally type EXIT which tells the SDK to run the Shutdown command, and therefore shuts down the GameServer.
```console
EXIT
```

If you run `kubectl describe gameserver` again - either the GameServer will be gone completely, or it will be in Shutdown state, on the way to being deleted.







