# Build The Infrastructure Needed for Today's Exercises

## Required Tools

### Cloud Shell
The entire lab can be completed in bash flavored [Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview). If you're not familiar with Cloud Shell, or how to set it up, you are probably in the wrong lab.

### Local Machine (Optional)
If cloud shell timeouts get your ire up, feel free to install these tools and run the lab locally:  
* Bash
* Docker
* Azure CLI
* Visual Studio Code
* Helm
* Kubernetes CLI (kubectl)
* git tools
* jq - tool for querying json objects in bash [link](https://stedolan.github.io/jq/)

> **NOTE:**
> Windows users beware! You might want to use ~~powershell~~ bash flavored cloud shell as some of the tools are difficult to get running on Win10.

## AKS Deployment
For this lab, we will install a vanilla 3-node AKS cluster that is RBAC enabled.

```console
GROUP_NAME=k8s_appdev_adv
AKS_NAME=<aks-name>
az group create -n $GROUP_NAME -l <azure-region>
az aks create -n $AKS_NAME -g $GROUP_NAME -k 1.11.6 --generate-ssh-keys 
```
Once your instance of AKS is turned up, use azure cli to pull down a kubeconfig context that will give you access to the api.
```console
az aks get-credentials -n $AKS_NAME -g $GROUP_NAME
```
Validate that you can call the AKS api and that the AKS nodes are ready by using kubectl get to check the status of the nodes.
```console
kubectl get nodes
```
With your eyeballs, check the ready status of each node in the response which should look something like this:
```output
NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-42628860-0   Ready     agent     3h        v1.11.6
aks-nodepool1-42628860-1   Ready     agent     3h        v1.11.6
aks-nodepool1-42628860-2   Ready     agent     3h        v1.11.6
```
That will do it, now you are "Ready", pun intended, for the sultry sounds of Eddie V's voice.



    
