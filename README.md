Instructions to deploy Istio and Kiali on a running Kubernetes cluster.

## Prerequisites

* A working Kubernetes cluster.
* (`kubectl`)[https://kubernetes.io/docs/tasks/tools/install-kubectl/] installed in the `PATH` with a working configuration.
* Admin role on the Kubernetes cluster.
* (`helm`)[https://github.com/helm/helm/releases] client installed in the `PATH`.

## Steps

### Download the Istio release

```bash
curl -L https://git.io/getLatestIstio | sh -
cd istio-1.0.5
export PATH=$PWD/bin:$PATH
```

### Install Istio CRDs

```bash
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
```

### Deploy Istio and Kiali

This uses the Helm client to render the Kubernetes manifest locally.

```bash
helm template install/kubernetes/helm/istio --name istio \
    # The namespace where Istio services will be created.
    --namespace istio-system \
    # LoadBalancer requires a Kubernetes cluster that supports this flavor.
    # Set it to NodePort otherwise.
    --set gateways.enabled=true --set gateways.istio-ingressgateway.type=LoadBalancer \
    # You can change the passphrase to whatever you want.
    --set kiali.enabled=true --set kiali.dashboard.passphrase=supersecret --set kiali.tag=v0.12 \
    --set prometheus.tag=v2.5.0 \
    --set grafana.enabled=true --set grafana.image.tag=5.4.2 \
    --set tracing.enabled=true \
    > /tmp/istio.yml
kubectl create namespace istio-system
kubectl apply -f /tmp/istio.yml
```

### Check the deployment

```bash
kubectl config set-context --current --namespace=istio-system
kubectl get pods
```

All pods should be either `Running` or `Completed`.

```
NAME                                      READY     STATUS      RESTARTS   AGE
grafana-65bfcb7f7b-bmrx7                  1/1       Running     0          153m
istio-citadel-856f994c58-l5z7j            1/1       Running     0          153m
istio-cleanup-secrets-4dhgt               0/1       Completed   0          153m
istio-egressgateway-5649fcf57-4rz6m       1/1       Running     0          153m
istio-galley-7665f65c9c-6596c             1/1       Running     0          153m
istio-grafana-post-install-h5k9s          0/1       Completed   0          153m
istio-ingressgateway-6755b9bbf6-rhcvl     1/1       Running     0          153m
istio-pilot-56855d999b-5wqcm              2/2       Running     0          153m
istio-policy-6fcb6d655f-txwl5             2/2       Running     0          153m
istio-security-post-install-fh8z4         0/1       Completed   0          153m
istio-sidecar-injector-768c79f7bf-hczjm   1/1       Running     0          153m
istio-telemetry-664d896cf5-rw8wg          2/2       Running     0          153m
istio-tracing-6b994895fd-f4j8j            1/1       Running     0          153m
kiali-67c69889b5-ljsmm                    1/1       Running     0          153m
prometheus-5b8d8fcbdc-xzjzx               1/1       Running     0          149m
```

### Access the Kiali UI

```bash
kubectl port-forward $(kubectl get pods -l app=kiali -o jsonpath='{ .items[0].metadata.name }') 20001
```

Then go to http://localhost:20001/ to access the Kiali dashboard.

### Deploy the bookinfo application

```bash
kubectl create namespace bookinfo
kubectl label namespace bookinfo istio-injection=enabled
kubectl apply -n bookinfo -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -n bookinfo -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

