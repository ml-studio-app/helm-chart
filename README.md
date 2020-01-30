# Prerequisites

## A cluster

### Minikube
https://minikube.sigs.k8s.io

### Docker Desktop
https://www.docker.com/products/docker-desktop

### GKE cluster
```shell script
export cluster_name=mlstudio-cluster
export cluster_zone=us-central1-a

gcloud container clusters create $cluster_name \
    --machine-type=n1-standard-4 \
    --num-nodes 2 \
    --enable-autoscaling --min-nodes 0 --max-nodes 6 \
    --zone $cluster_zone

# Set kubectl locally
gcloud container clusters get-credentials $cluster_name --zone $cluster_zone
```

## Kubectl
export KUBE_VERSION=v1.17.0
export KUBE_OS=darwin # windows # linux
wget -q https://storage.googleapis.com/kubernetes-release/release/${KUBE_VERSION}/bin/${KUBE_OS}/amd64/kubectl -O /usr/local/bin/kubectl && \
chmod +x /usr/local/bin/kubectl

## Helm 2.x
```shell script
# TODO :: Install helm 2.x if not installed, https://helm.sh/docs/install/#installing-helm
export HELM_VERSION=v2.16.0 # v3.0.2
export HELM_OS=darwin # windows # linux
wget -q https://get.helm.sh/helm-${HELM_VERSION}-${HELM_OS}-amd64.tar.gz -O - | tar -xzO ${HELM_OS}-amd64/helm > /usr/local/bin/helm && \
chmod +x /usr/local/bin/helm

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

# TODO :: Wait for teller to get ready.
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

# If you are trying this on GKE then skip deploying the metrics server because it comes with GKE 
helm install mlstudio/mlstudio --set metrics-server.enabled=false
# Otherwise
helm install mlstudio/mlstudio
```

# Find the IP to access the app
```shell script
kubectl get svc istio-ingressgateway -n istio-system
```

# Log in as an admin user
```shell script
  USER: admin
  PASSWORD: password
```

# Cleanup
```shell script
# Delete ML Studio release
helm del --purge mlstudio

# Delete the GKE cluster
gcloud container clusters delete $cluster_name --zone $cluster_zone
```