# Pre-Requisites

* K8s cluster
* Helm and Kubectl configured
* ArgoCD deployed 
* Istio deployed
* Prometheus deployed

** K8s Cluster - AKS **
```
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.ContainerService
az provider show -n Microsoft.OperationalInsights
az provider show -n Microsoft.ContainerService
az aks create --resource-group mario-aks-auckland-meetup-2026-demo --name mario-aks-auckland-demo-aks --node-count 1 --enable-addons monitoring --generate-ssh-keys

az aks list --output table
az aks get-credentials --resource-group mario-aks-auckland-meetup-2026-demo --name mario-aks-auckland-demo-aks

kubectl config get-contexts

```

** Install Istio Service Mesh
```
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
kubectl create namespace istio-system
helm install istio-base istio/base  -n istio-system
kubectl get crds | grep istio
helm install istiod istio/istiod -n istio-system
kubectl get pods -n istio-system | grep -i istiod
helm install istio-ingress istio/gateway -n istio-system
kubectl get svc -n istio-system | grep -i ingress
```

** Enable sidecar injections for the pods in the demo namespace
```
kubectl create namespace demo
kubectl label namespace demo istio-injection=enabled
kubectl get ns demo --show-labels
kubectl run test --image=nginx -n demo
kubectl get pod test -n demo -o jsonpath='{.spec.containers[*].name}'
kubectl get svc -n istio-system
kubectl port-forward svc/istio-ingress -n istio-system 8080:80
```

** Install ArgoCD and Argo Rollouts
```
kubectl create namespace argo-rollouts

kubectl apply -n argo-rollouts -f \
https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

kubectl get pods -n argo-rollouts

brew install argoproj/tap/kubectl-argo-rollouts
kubectl argo rollouts version
kubectl argo rollouts get rollout demo-app -w

kubectl create namespace argocd
kubectl apply -n argocd -f \
https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


kubectl get pods -n argocd

kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d

kubectl port-forward svc/argocd-server -n argocd 8080:443

username: admin
password: <output>
```

** Create ArgoCD application

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd

spec:

  project: default

  source:
    repoURL: https://github.com/mri-fernando/argo-canary-demo-helm
    targetRevision: HEAD
    path: demo-app

  destination:
    server: https://kubernetes.default.svc
    namespace: demo

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

  ignoreDifferences:
    - group: networking.istio.io
      kind: VirtualService
      jsonPointers:
        - /spec/http/0/route

```

** Apply the YAML (In realworld this could be a CI CD pipeline doing it)
```
kubectl apply -f application.yaml
```

** Verify RolloutWorks
```
kubectl get rollouts -n demo

kubectl argo rollouts get rollout demo-app -n demo -w

```

** RollOuts Dashboard
```
kubectl argo rollouts dashboard
```

** Troubleshooting
```
kubectl logs -n argo-rollouts deploy/argo-rollouts -f
```

** Access the APP

*** Using the k8s service will not loadbalance as we're bypassing the Istio VirtualService, so it will always be going to the same set of pods
```
kubectl port-forward svc/demo-app -n demo 8081:80
curl -vvv http://localhosts:8081
```


*** using Istio Ingress Gateway
```
kubectl port-forward svc/istio-ingress -n istio-system 8081:80
```

Get the public IP
```
 kubectl -n istio-system get svc istio-ingress -o yaml 

 ```

This will return a mix of traffic to both stable and canary
```
while True; do curl -H "Host: mario-app-demo.mario.com" http://4.147.48.56; sleep 1; echo -e "\n\n"; done 
```








