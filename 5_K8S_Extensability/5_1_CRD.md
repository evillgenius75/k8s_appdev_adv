# Extending Kubernetes with Custom Resources

Let’s assume you want to build a clustered application or a software as a service offering. Before you start to write one single line of application code, you must address an overflowing array of architectural issues, including security, multitenancy, API gateways, CLI, configuration management, and logging.

What if you could just leverage all that infrastructure from Kubernetes, save yourself few man years in development, and focus on implementing your unique service?

The latest Kubernetes adds an important feature called CustomResourceDefinitions (CRD), which enables plugging in your own managed object and application as if it were a native Kubernetes component. This way, you can leverage Kubernetes CLI, API services, security and cluster management frameworks without modifying Kubernetes or knowing its internals.

Since this is our last lab let's have some fun! Let's make a Game Server using Kubernetes with Project Agones!

## Allowing UDP traffic
For Agones to work correctly, we need to allow UDP traffic to pass through to our AKS cluster. To achieve this, we must update the NSG (Network Security Group) with the proper rule. A simple way to do that is:

Login to the Azure Portal
Find the resource group where the AKS resources are kept, which should have a name like MC_resourceGroupName_AKSName_westeurope. Alternative, you can type az resource show --namespace Microsoft.ContainerService --resource-type managedClusters -g $AKS_RESOURCE_GROUP -n $AKS_NAME -o json | jq .properties.nodeResourceGroup
Find the Network Security Group object, which should have a name like aks-agentpool-********-nsg
Select Inbound Security Rules
Select Add to create a new Rule with UDP as the protocol and 7000-8000 as the Destination Port Ranges. Pick a proper name and leave everything else at their default values
Alternatively, you can use the following command, after modifying the RESOURCE_GROUP_WITH_AKS_RESOURCES and NSG_NAME values:

```console
az network nsg rule create \
  --resource-group RESOURCE_GROUP_WITH_AKS_RESOURCES \
  --nsg-name NSG_NAME \
  --name AgonesUDP \
  --access Allow \
  --protocol Udp \
  --direction Inbound \
  --priority 520 \
  --source-port-range "*" \
  --destination-port-range 7000-8000
  ```

Creating and assigning Public IPs to Nodes
Nodes in AKS don’t get a Public IP by default. To assign a Public IP to a Node, find the Resource Group where the AKS resources are installed on the portal (it should have a name like MC_resourceGroupName_AKSName_westeurope). Run the following yaml to create a DaemonSet that will automatically assign a PublicIP to each node of your cluster.

```console
kubectl create -n kube-system -f https://raw.githubusercontent.com/dgkanatsios/AksNodePublicIPController/master/deploy.yaml
```
