# Prerequisites

## Helm 2.x
```shell script
# Install helm 2.x if not installed, https://helm.sh/docs/install/#installing-helm

# Setup Tiller Service account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF

helm init --service-account=tiller
helm repo update

# Ensure that tiller is secure from access inside the cluster
kubectl patch deployment tiller-deploy --namespace=kube-system --type=json --patch='[{"op": "add", "path": "/spec/template/spec/containers/0/command", "value": ["/tiller", "--listen=localhost:44134"]}]'

# Verify helm and tiller were installed properly, by checking the client and server versions
helm version --short

# Wait for teller to get ready.
```

## Istio
```shell script
curl -L https://git.io/getLatestIstio | sh -

cd istio-*/

# Install the istio-init chart to bootstrap all the Istio’s CRDs:
helm install install/kubernetes/helm/istio-init --name istio-init --namespace istio-system

# Wait for all Istio CRDs to be created:
kubectl -n istio-system wait --for=condition=complete job --all
```

```shell script
# Select a configuration profile and then install the istio chart corresponding to your chosen profile. The default profile is recommended for production deployments
helm install install/kubernetes/helm/istio --name istio --namespace istio-system

# Allow automatic Istio sidecar injector on all containers in default namespace
kubectl label namespace default istio-injection=enabled

# Verify auto injection worked
kubectl get namespace -L istio-injection

# Back
cd ../

# Remove istio directory
rm -R istio-*/
```

# Install
```shell script
helm repo add mlstudio https://ml-studio-app.github.io/helm-chart/
helm repo update

helm install mlstudio/mlstudio
```
