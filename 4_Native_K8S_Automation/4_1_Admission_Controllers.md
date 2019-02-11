# Admission Controllers

An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized. The controllers are compiled into the kube-apiserver binary, and may only be configured by the cluster administrator. 

There are two special controllers: 
* MutatingAdmissionWebhook 
* ValidatingAdmissionWebhook. 

These execute the mutating and validating (respectively) admission control webhooks which are configured in the API.

Admission controllers may be “validating”, “mutating”, or both. Mutating controllers may modify the objects they admit, validating controllers may not.

The admission control process proceeds in two phases. In the first phase, mutating admission controllers are run. In the second phase, validating admission controllers are run.  

> NOTE: A controller instance can participate in both phases.

If any of the controllers in either phase reject the request, the entire request is rejected immediately and an error is returned to the end-user.

Finally, in addition to sometimes mutating the object in question, admission controllers may sometimes have side effects, that is, mutate related resources as part of request processing. Incrementing quota usage is the canonical example of why this is necessary. Any such side-effect needs a corresponding reclamation or reconciliation process, as a given admission controller does not know for sure that a given request will pass all of the other admission controllers.

### MutatingAdmissionWebhook
This admission controller (as implied by the name) only runs in the mutating phase. It calls any mutating webhooks which match the request. Matching webhooks are called serially; each one may modify the object if it desires.

If a webhook called by this controller has side effects (for example, decrementing quota) it must have a reconciliation system, as it is not guaranteed that subsequent webhooks or validating admission controllers will permit the request to finish.

If you disable the MutatingAdmissionWebhook, you must also disable the MutatingWebhookConfiguration object in the ```admissionregistration.k8s.io/v1beta1 group/version``` via the ```--runtime-config``` flag (both are on by default in versions >= 1.9).

* Use caution when authoring and installing mutating webhooks
* Users may be confused when the objects they try to create are different from what they get back.
* Built in control loops may break when the objects they try to create are different when read back.
* Setting originally unset fields is less likely to cause problems than overwriting fields set in the original request. Avoid doing the latter.
* This is a beta feature. Future versions of Kubernetes may restrict the types of mutations these webhooks can make.
* Future changes to control loops for built-in resources or third-party resources may break webhooks that work well today. Even when the webhook installation API is finalized, not all possible webhook behaviors will be guaranteed to be supported indefinitely.

### ValidatingAdmissionWebhook
This admission controller only runs in the validation phase. The webhooks it calls may __not__ mutate the object. (This is different than webhooks called by the MutatingAdmissionWebhook admission controller.) It calls any validating webhooks which match the request. Matching webhooks are called in parallel, and if any of them rejects the request, then the request fails. 

If a webhook called by this has side effects (for example, decrementing quota) it must have a reconciliation system, as it is not guaranteed that subsequent webhooks or other validating admission controllers will permit the request to finish.

If you disable the ValidatingAdmissionWebhook, you must also disable the ValidatingWebhookConfiguration object in the admissionregistration.k8s.io/v1beta1 group/version via the --runtime-config flag (both are on by default in versions 1.9 and later).

## Lab 4.1 - Use a MuatatingAdmissionWebhook to make all services tht are of type LoadBalancer to be Internal type

Clone the following repo for the AdmissionController Helm Chart

```console
git clone https://github.com/evillgenius75/internallb-webhook-admission-controller.git

cd internallb-webhook-admission-controller
```

Deploy the Helm Chart to your cluster

```console
helm install --name admission-webhook charts/internallb-webhook-admission-controller
```

This will create a pod called `admission-webhook-internallb-webhook-admission-controller-*`
Verify the Pod is running

```console
kubectl get pods
```

Next, create the following deployment and service manifest.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: foo-svc
  labels:
    app: foo-svc
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: foo-http
  selector:
    app: foo-svc
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: foo-svc
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: foo-svc
    spec:
      containers:
      - name: foo-svc
        image: gcr.io/google_containers/echoserver:1.5
        ports:
        - containerPort: 8080
        env:
           - name: NODE_NAME
             valueFrom:
               fieldRef:
                 fieldPath: spec.nodeName
           - name: POD_NAME
             valueFrom:
               fieldRef:
                 fieldPath: metadata.name
           - name: POD_NAMESPACE
             valueFrom:
               fieldRef:
                 fieldPath: metadata.namespace
           - name: POD_IP
             valueFrom:
               fieldRef:
                 fieldPath: status.podIP
```

Deploy it.

```console
kubectl apply -f admission.yaml
```

Notice that the Service type is LoadBalancer which will normally create a Public Facing Load Balancer. Because the Admission Controller was created in *Mutating* mode when the Service is deployed, it will be mutated to automatically add the correct annotation to make the LoadBalancer an Azure Internal LoadBalancer cloud resource.

```console
kubectl get svc -w
```
```output
NAME                                                        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
admission-webhook-internallb-webhook-admission-controller   NodePort       172.23.217.199   <none>        443:32513/TCP   1h
foo-svc                                                     LoadBalancer   172.23.161.50    172.22.0.97   80:32545/TCP    1h
kubernetes                                                  ClusterIP      172.23.0.1       <none>        443/TCP         23d
```

You will notice that at some point the External IP of the foo-svc service will have an IP address on the Subnet Address space of your AKS cluster and not a Public IP. 

You can also get the description of the service and see the proper annotation was created.

```console
kubectl describe svc foo-svc
```
```output
Name:                     foo-svc
Namespace:                default
Labels:                   app=foo-svc
Annotations:              service.beta.kubernetes.io/azure-load-balancer-internal: true
Selector:                 app=foo-svc
Type:                     LoadBalancer
IP:                       172.23.161.50
LoadBalancer Ingress:     172.22.0.97
Port:                     foo-http  80/TCP
TargetPort:               8080/TCP
NodePort:                 foo-http  32545/TCP
Endpoints:                172.22.0.16:8080,172.22.0.61:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  EnsuringLoadBalancer  64m   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   63m   service-controller  Ensured load balancer
```

Now your policy is applied automatically.
