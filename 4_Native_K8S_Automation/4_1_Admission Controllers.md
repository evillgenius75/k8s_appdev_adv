# Admission Controllers

Kubernetes is highly configurable and extensible. As a result, there is rarely a need to fork or submit patches to the Kubernetes project code.

Kubernetes is designed to be automated by writing client programs. Any program that reads and/or writes to the Kubernetes API can provide useful automation. Automation can run on the cluster or off it. By following the guidance in this doc you can write highly available and robust automation. Automation generally works with any Kubernetes cluster, including hosted clusters and managed installations.

There is a specific pattern for writing client programs that work well with Kubernetes called the Controller pattern. Controllers typically read an object’s .spec, possibly do things, and then update the object’s .status.

A controller is a client of Kubernetes. When Kubernetes is the client and calls out to a remote service, it is called a Webhook. The remote service is called a Webhook Backend. Like Controllers, Webhooks do add a point of failure.

In the webhook model, Kubernetes makes a network request to a remote service. In the Binary Plugin model, Kubernetes executes a binary (program). Binary plugins are used by the kubelet (e.g. Flex Volume Plugins and Network Plugins) and by kubectl.