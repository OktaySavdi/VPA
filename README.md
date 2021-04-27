### Required
- Metric Server

### VPA

- It requires at least two healthy pod replicas to work
- As Vertical Pod Autoscaling modifies the requests and limits automatically, you cannot use it with a Horizontal Pod Autoscaler because HPA relies on the CPU and Memory utilisation to horizontally scale pods.
- VPA allocates a minimum memory of 250MiB regardless of what you specify. Though this default can be modified at a global level
 - VPA only works with Deployments, StatefulSets, DaemonSets,ReplicaSets
   etc. You cannot use it with a standalone Pod that does not have an
   owner.

### Components of VPA

The project consists of 3 components:

-   [Recommender](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/recommender/README.md)  - checks for historical resource utilisation and current usage patterns and recommends an ideal resource request value.
    
-   [Updater](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/updater/README.md)  - it checks which of the managed pods have correct resources set and, if not, kills them so that they can be recreated by their controllers with the updated requests.
    
-   [Admission Plugin](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/admission-controller/README.md)  - it sets the correct resource requests on new pods (either just created or recreated by their controller due to Updater's activity).

### Example VPA configuration

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       my-app
  updatePolicy:
    updateMode: "Auto"
```

### VPA Manifest

Let’s look at an example VPA Manifest below:

Like any manifest, it has an  `apiVersion`,  `kind`,  `metadata`, and  `spec`  section.

Within the spec section, we see a  `targetRef`  section that specifies the object that this VPA applies to. The  `updatePolicy`  section defines whether this VPA would recommend or recommend and autoscale based on the  `updateMode`  property. If the  `updateMode`  property is set to  `Auto`, it autoscales pods vertically, if  `Off`, it merely recommends the ideal resource request values.

The  `resourcePolicy`  section allows us to specify  `containerPolicies`  for each container with a  `minAllowed`  and  `maxAllowed`  resource values. That is extremely important to save yourself from memory leaks. You can also switch off resource recommendations and autoscaling on a particular container within a pod, typically seen with Istio sidecars or  `InitContainers`.

### Quick start

After  [installation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#installation)  the system is ready to recommend and set resource requests for your pods. In order to use it you need to insert a  _Vertical Pod Autoscaler_  resource for each controller that you want to have automatically computed resource requirements. This will be most commonly a  **Deployment**. There are three modes in which  _VPAs_  operate:

-   `"Auto"`: VPA assigns resource requests on pod creation as well as updates them on existing pods using the preferred update mechanism. Currently this is equivalent to  `"Recreate"`  (see below). Once restart free ("in-place") update of pod requests is available, it may be used as the preferred update mechanism by the  `"Auto"`  mode.  **NOTE:**  This feature of VPA is experimental and may cause downtime for your applications.

-   `"Recreate"`: VPA assigns resource requests on pod creation as well as updates them on existing pods by evicting them when the requested resources differ significantly from the new recommendation (respecting the Pod Disruption Budget, if defined). This mode should be used rarely, only if you need to ensure that the pods are restarted whenever the resource request changes. Otherwise prefer the  `"Auto"`  mode which may take advantage of restart free updates once they are available.  **NOTE:**  This feature of VPA is experimental and may cause downtime for your applications.
-   `"Initial"`: VPA only assigns resource requests on pod creation and never changes them later.
-   `"Off"`: VPA does not automatically change resource requirements of the pods. The recommendations are calculated and can be inspected in the VPA object.

###  Install command
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl edit -n kube-system deployments.apps metrics-server

- --kubelet-insecure-tls
```
```bash
git clone https://github.com/kubernetes/autoscaler.git
cd /root/autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

kubectl get po -n kube-system
```
### Test your installation

A simple way to check if Vertical Pod Autoscaler is fully operational in your cluster is to create a sample deployment and a corresponding VPA config:
```
kubectl create -f examples/hamster.yaml
```
The above command creates a deployment with 2 pods, each running a single container that requests 100 millicores and tries to utilize slightly above 500 millicores. The command also creates a VPA config pointing at the deployment. VPA will observe the behavior of the pods and after about 5 minutes they should get updated with a higher CPU request (note that VPA does not modify the template in the deployment, but the actual requests of the pods are updated). To see VPA config and current recommended resource requests run:
```
kubectl describe vpa
```
![image](https://user-images.githubusercontent.com/3519706/116238558-27bcf900-a76a-11eb-822e-0ce7419fb5e7.png)

If you look at the recommendation section, you’ll see several recommendations.

-   **Target**: This is the real value that VPA will use when it evicts the current pod and create another one. If you are scraping these metrics, you should always track the target value.
-   **Lower Bound** : This reflects the lower bound of triggering a resize. If your pod utilisation goes below this, VPA will evict it and scale it down.
-   **Upper Bound** : This denotes the upper bound for the next resizing to trigger. If you pod utilisation goes above these values, VPA will evict it and scale it up.
-   **Uncapped target**: This denotes the target utilisation if you didn’t provide a minimum or maximum boundary to the VPA.


### Troubleshooting

To diagnose problems with a VPA installation, perform the following steps:

-   Check if all system components are running:
```
kubectl --namespace=kube-system get pods|grep vpa
```

The above command should list 3 pods (recommender, updater and admission-controller) all in state Running.

-   Check if the system components log any errors. For each of the pods returned by the previous command do:
```
kubectl --namespace=kube-system logs [pod name]| grep -e '^E[0-9]\{4\}'
```
-   Check that the VPA Custom Resource Definition was created:
```
kubectl get customresourcedefinition|grep verticalpodautoscalers
```

### Reference

[1] https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler

[2] https://betterprogramming.pub/understanding-vertical-pod-autoscaling-in-kubernetes-6d53e6d96ef3

[3] https://medium.com/infrastructure-adventures/vertical-pod-autoscaler-deep-dive-limitations-and-real-world-examples-9195f8422724 
