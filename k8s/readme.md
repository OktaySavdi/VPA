## Installing the Vertical Pod Autoscaler 

You can use the Kubernetese to install the Vertical Pod Autoscaler (VPA).

**Install Metric Server**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl edit -n kube-system deployments.apps metrics-server

- --kubelet-insecure-tls
```
**Install VPA**
```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

kubectl get po -n kube-system
```
