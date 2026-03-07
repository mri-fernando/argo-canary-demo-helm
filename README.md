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
```


