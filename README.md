### Required
- Metric Server

### Highlight

![image](https://user-images.githubusercontent.com/3519706/116244974-24793b80-a771-11eb-9cd4-f832225a289d.png)

- It requires at least two healthy pod replicas to work (but with - --min-replicas = 1 you can also operate on a single pod.)
- Can suggest values or automatically update values for VPA, CPU and Memory requests and limits.
- VPA allocates a minimum memory of 250MiB regardless of what you specify. Though this default can be modified at a global level
- As Vertical Pod Autoscaling modifies the requests and limits automatically, you cannot use it with a Horizontal Pod Autoscaler because HPA relies on the CPU and Memory utilisation to horizontally scale pods.
- VPA only works with Deployments, StatefulSets, DaemonSets,ReplicaSets etc. You cannot use it with a standalone Pod that does not have an owner.
- Cluster nodes are used efficiently because pods use exactly what they need.
- VPA can adjust CPU and memory requests over time without you needing to do anything, reducing maintenance time.
- VPA supports max 500 VerticalPodAutoscaler objects
- HPA and VPA should not be used together for the same application.
- VPA min, max recommended values can be given #limitRange should be planned
- Metric server must be installed before
- The original Deployment specification will be untouched
But this has a nice side-effect: for example, if you’re using Argo CD, it won’t detect any diff in the Deployment specification.

### Components of VPA

The project consists of 3 components:

-   [Recommender]()  - Connects to the `metrics-server` application in the cluster, fetches historical and current usage data (CPU and memory) for each VPA-enabled pod and generates recommendations for scaling up or down the `requests` and `limits` of these pods.
    
-   [Updater]()  - Runs every 1 minute. If a pod is not running in the calculated recommendation range, it **evicts the currently running version of this pod**, so it can restart and go through the VPA admission webhook which will change the CPU and memory settings for it, before it can start.
    
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

### How it works

After  [installation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#installation)  the system is ready to recommend and set resource requests for your pods. In order to use it you need to insert a  _Vertical Pod Autoscaler_  resource for each controller that you want to have automatically computed resource requirements. This will be most commonly a  **Deployment**. There are three modes in which  _VPAs_  operate:

-   `"Auto"`: VPA assigns resource requests on pod creation as well as updates them on existing pods using the preferred update mechanism. Currently this is equivalent to  `"Recreate"`  (see below). Once restart free ("in-place") update of pod requests is available, it may be used as the preferred update mechanism by the  `"Auto"`  mode.  **NOTE:**  This feature of VPA is experimental and may cause downtime for your applications.

-   `"Recreate"`: VPA assigns resource requests on pod creation as well as updates them on existing pods by evicting them when the requested resources differ significantly from the new recommendation (respecting the Pod Disruption Budget, if defined). This mode should be used rarely, only if you need to ensure that the pods are restarted whenever the resource request changes. Otherwise prefer the  `"Auto"`  mode which may take advantage of restart free updates once they are available.  **NOTE:**  This feature of VPA is experimental and may cause downtime for your applications.
-   `"Initial"`: VPA only assigns resource requests on pod creation and never changes them later.
-   `"Off"`: VPA does not automatically change resource requirements of the pods. The recommendations are calculated and can be inspected in the VPA object.

### installattions

- install on Kubernetes - [URL](https://github.com/OktaySavdi/VPA/tree/main/k8s)
- install on Openshift  - [URL](https://github.com/OktaySavdi/VPA/tree/main/openshift)

### Test your installation

A simple way to check if Vertical Pod Autoscaler is fully operational in your cluster is to create a sample deployment and a corresponding VPA config:
```
git clone https://github.com/OktaySavdi/VPA.git
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
