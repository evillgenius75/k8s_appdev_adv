# Build the Infrastructure needed for todays Exercises

## Required Tools
This entire lab can be completed in cloud shell. If you don't know what cloudshell, or how to set it up, you are probably in the wrong lab.

If cloud shell timeouts get your ire up, feel free to install these tools and run the lab locally:  
* Bash
* Docker
* Azure CLI
* Visual Studio Code
* Helm
* Kubernetes CLI (kubectl)
* git tools
> **NOTE:**
> Windows users beware! You might want to use cloud shell as some of the tools are difficult to get running on Win10.

## AKS Deployment
For this lab, we will install a vanilla 3 node AKS cluster that is RBAC enabled.
```console
az group create -n <group-name> -l <azure-region>
az aks create -n <aks-name> -g <group-name>
```
Once your instance of AKS is turned up, use azure cli to pull down a kubeconfig context that will give you access to the api.
```console
az aks get-credentials -n <aks-name> -g <aks-group>
```
Validate that you can call the AKS api and that the AKS nodes are ready by using kubectl get to check the status of the nodes.
```console
kubectl get nodes
```
With your eyeballs, check the ready status of each node in the response which should look something like this.
```output
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-38423990-vmss000000   Ready    agent   29d   v1.11.5
aks-nodepool1-38423990-vmss000001   Ready    agent   29d   v1.11.5
aks-nodepool1-38423990-vmss000002   Ready    agent   29d   v1.11.5
```
That will do it, now you are "Ready" - pun intended - for the sultry sounds of Eddie V's voice.



    