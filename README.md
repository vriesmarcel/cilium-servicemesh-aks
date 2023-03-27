# This repo is documentation of what I did at our innovation day 3/27/2022 to get cilium running on AKS.
# steps to get Cilium running on AKS

Docs I used come from https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/

## Get started with a cluster that has Bring Your Own CNI


az group create --name "aks-cilium-rg" -l westus2

## Create AKS cluster
az aks create --resource-group "aks-cilium-rg" --name "aks-cilium" --network-plugin none

## Get the credentials to access the cluster with kubectl
az aks get-credentials --resource-group "aks-cilium-rg" --name "aks-cilium"

## Download the Cilium Command Line Interface (CLI)
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/v0.13.2/cilium-windows-amd64.tar.gz{,.sha256sum}

## Unpack the Tarball
 tar xzvfC cilium-windows-amd64.tar.gz c:/dapr

## install cilium in the cluster that is in the current context of kubectl
cilium install --azure-resource-group aks-cilium-rg

Results:
``` cmd
C:\Users\vries>cilium install --azure-resource-group aks-cilium-rg
🔮 Auto-detected Kubernetes kind: AKS
✨ Running "AKS" validation checks
✅ Detected az binary
ℹ️  Using Cilium version 1.13.1
🔮 Auto-detected cluster name: aks-cilium
✅ Derived Azure subscription ID e57087a4-a053-48c9-8857-995316398cdc from subscription MVPSponsorship
✅ Detected Azure AKS cluster in BYOCNI mode (no CNI plugin pre-installed)
🔮 Auto-detected datapath mode: aks-byocni
🔮 Auto-detected kube-proxy has been installed
⚠️ Unable to list kubernetes api resources, try --api-versions if needed: failed to list api resources: unable to retrieve the complete list of server APIs: metrics.k8s.io/v1beta1: the server is currently unable to handle the request
ℹ️  helm template --namespace kube-system cilium cilium/cilium --version 1.13.1 --set aksbyocni.enabled=true,azure.resourceGroup=aks-cilium-rg,cluster.id=0,cluster.name=aks-cilium,encryption.nodeEncryption=false,kubeProxyReplacement=disabled,nodeinit.enabled=true,operator.replicas=1,serviceAccounts.cilium.name=cilium,serviceAccounts.operator.name=cilium-operator
ℹ️  Storing helm values file in kube-system/cilium-cli-helm-values Secret
🔑 Created CA in secret cilium-ca
🔑 Generating certificates for Hubble...
🚀 Creating Service accounts...
🚀 Creating Cluster roles...
🚀 Creating ConfigMap for Cilium version 1.13.1...
🚀 Creating AKS Node Init DaemonSet...
🚀 Creating Agent DaemonSet...
🚀 Creating Operator Deployment...
⌛ Waiting for Cilium to be installed and ready...
✅ Cilium was successfully installed! Run 'cilium status' to view installation health

C:\Users\vries>cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:          OK
 \__/¯¯\__/    Operator:        OK
 /¯¯\__/¯¯\    Hubble Relay:    disabled
 \__/¯¯\__/    ClusterMesh:     disabled
    \__/

Deployment        cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet         cilium             Desired: 3, Ready: 3/3, Available: 3/3
Containers:       cilium             Running: 3
                  cilium-operator    Running: 1
Cluster Pods:     6/6 managed by Cilium
Image versions    cilium             quay.io/cilium/cilium:v1.13.1@sha256:428a09552707cc90228b7ff48c6e7a33dc0a97fe1dd93311ca672834be25beda: 3
                  cilium-operator    quay.io/cilium/operator-generic:v1.13.1@sha256:f47ba86042e11b11b1a1e3c8c34768a171c6d8316a3856253f4ad4a92615d555: 1
```
## install hubble observabillity & commandline
Install hubble including the UI so you can visualize trafic.

```cmd 
cilium hubble enable --ui
```

```cmd
curl -LO "https://raw.githubusercontent.com/cilium/hubble/master/stable.txt"
set /p HUBBLE_VERSION=<stable.txt
curl -L --fail -O "https://github.com/cilium/hubble/releases/download/%HUBBLE_VERSION%/hubble-windows-amd64.tar.gz"
curl -L --fail -O "https://github.com/cilium/hubble/releases/download/%HUBBLE_VERSION%/hubble-windows-amd64.tar.gz.sha256sum"
certutil -hashfile hubble-windows-amd64.tar.gz SHA256
tar zxf hubble-windows-amd64.tar.gz
```

At other command prompt run: 
```
cilium hubble port-forward&
```
And

```
hubble status
```

result:
```cmd
Healthcheck (via localhost:4245): Ok
Current/Max Flows: 10,168/12,285 (82.77%)
Flows/s: 20.81
Connected Nodes: 3/3
```
## Add the Huble UI to the cluster so we can visualize trafic

start the UI
```
cilium hubble ui
```

## Now test the results of cilium
Run application and generate load and see if we have results

kubectl apply -f catalog.yaml
kubectl apply -f ordering.yaml
kubectl apply -f frontend.yaml
kubectl apply -f sqlserver.yaml

## set the cilium ingress
helm upgrade cilium cilium/cilium --version 1.13  --namespace kube-system --reuse-values --set ingressController.enabled=true --set ingressController.loadbalancerMode=dedicated

