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

# Get URL for ArgoCD
k get gateway -n demo -o yaml

kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d

kubectl port-forward svc/argocd-server -n argocd 8080:443

username: admin
password: <output>
```

** Install Prometheus using helm chart

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set alertmanager.enabled=false \
  --create-namespace 
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

** Verify Rollout Works
```
kubectl get rollouts -n demo

kubectl argo rollouts get rollout demo-app -n demo -w

```

** OPTIONAL: RollOuts Dashboard
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

Update /etc/hosts

```
sudo vi /etc/hosts
4.147.48.56 demo-app.mario.com
4.147.48.56 argo-demo.mario.com
4.147.48.56 prometheus.mario.com
```

** Access ArgoCD, Prometheus and App UI
Access App: http://demo-app.mario.com/query 
Access ArgoCD: http://argo-demo.mario.com/query 
Access Prometheus: http://prometheus.mario.com/query 


**Demonstrate manual promote
*** Bump app.py and rollout a new version 

Now bump the values.yaml to following
```
rollout:
  steps:
  - setWeight: 10
  - pause: {}  # Waits for manual promotion
  - setWeight: 50
  - pause: {duration: 5m}

```

** This will return a mix of traffic to both stable and canary
```
while True; do curl http://demo-app.mario.com; echo -e "\nrequest sent"; echo -e "\n\n"; done
```


** Manually promote the canary 
```
kubectl argo rollouts promote demo-app -n demo

# If something goes wrong
kubectl argo rollouts abort demo-app -n demo

```

** Demo Istio Prometheus metrics 

``` 
kubectl get servicemonitor -n monitoring -o yaml 
kubectl port-forward po/demo-app-545b8b555b-b44nk -n demo 15090:15090
curl http://localhost:15090/stats/prometheus
```

** Display the prometheus metrics in the UI 

Send some requests to http://demo-app.mario.com/

http://prometheus.mario.com/query 
Metric: istio_requests_total

Monitor the prometheus UI
http://prometheus.mario.com/

```
          sum(rate(istio_requests_total{
            reporter="destination",
            destination_service=~"demo-app-canary.demo.svc.cluster.local",
            destination_workload_namespace="demo",
            response_code!~"5.*"
          }[2m])) 
          /
          sum(rate(istio_requests_total{
            reporter="destination",
            destination_service=~"demo-app-canary.demo.svc.cluster.local",
            destination_workload_namespace="demo"
          }[2m]))
```

** Automated Promotion Demo

Keep this running in the background
while True; do curl http://demo-app.mario.com; echo -e "\nrequest sent"; echo -e "\n\n"; done

Bump values.yaml in rollout block
Sync ArgoCD application 


** Automated RollBack Demo
** Bump app.py and push and Github actions should rollout a buggy version
https://github.com/mri-fernando/argo-canary-demo-app/actions


Sync ArgoCD application, this should deploy a new one


Now monitor the rollout

```
kubectl argo rollouts get rollout demo-app -n demo -w
kubectl get virtualservice  demo-app -n demo -o yaml 
kubectl describe AnalysisTemplate -n demo

```

Monitor the prometheus UI
http://prometheus.mario.com/

```
          sum(rate(istio_requests_total{
            reporter="destination",
            destination_service=~"demo-app-canary.demo.svc.cluster.local",
            destination_workload_namespace="demo",
            response_code!~"5.*"
          }[2m])) 
          /
          sum(rate(istio_requests_total{
            reporter="destination",
            destination_service=~"demo-app-canary.demo.svc.cluster.local",
            destination_workload_namespace="demo"
          }[2m]))
```



** Demonstrate how Blue Green works with Argo Rollouts




# TODO

Blue Green demo completion
Git pull issues with argo-canary-demo-helm repo
move unwanted yaml out of templates to a separate directory in argo-canary-demo-helm repo
Presentation
Nice diagrams
Github actions credits
Azure credits






