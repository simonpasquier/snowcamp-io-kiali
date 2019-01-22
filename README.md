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
# LoadBalancer requires a Kubernetes cluster that supports this flavor. Set it to NodePort otherwise.
# You can change the Kiali passphrase to whatever you want.
# This enables tracing with 25% of requests sampled.
helm template install/kubernetes/helm/istio --name istio \
    --namespace istio-system \
    --set gateways.enabled=true --set gateways.istio-ingressgateway.type=LoadBalancer \
    --set kiali.enabled=true --set kiali.dashboard.passphrase=supersecret --set kiali.tag=v0.12 \
    --set prometheus.tag=v2.5.0 \
    --set grafana.enabled=true --set grafana.image.tag=5.4.2 \
    --set tracing.enabled=true --set pilot.traceSampling=25.0 \
    > /tmp/istio.yml
kubectl create namespace istio-system
kubectl apply -f /tmp/istio.yml
# Delete the default ingress gateway that we won't use anyway.
kubectl delete gateways/istio-autogenerated-k8s-ingress
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

In a separate Shell window, run:


```bash
kubectl port-forward svc/kiali 20001
```

Then go to http://localhost:20001/ to access the Kiali dashboard.

### Deploy the httpbin application

Lets deploy a very simple application composed of only one service.

Run the commands from this project's directory.

```bash
kubectl create namespace httpbin
# Enable auto-injection of the proxy sidecar
kubectl label namespace httpbin istio-injection=enabled
kubectl apply -n httpbin -f ./httpbin/
```

```bash
curl -i http://httpbin.snowcamp.example.com/headers
curl -i http://httpbin.snowcamp.example.com/status/200
```

### Deploy the bookinfo application

This deploys a more complex application composed of 3 microservics. The initial revision will direct traffic to the `v1` of each microservice.

Run the commands from this project's directory.

```bash
kubectl create namespace bookinfo
# Enable auto-injection of the proxy sidecar
kubectl label namespace bookinfo istio-injection=enabled
kubectl apply -f ./bookinfo/01_initial_setup/
```

### Update the details service

TBC

### Access the other dashboards

In a separate Shell windows, run:

```bash
kubectl port-forward -n istio-system svc/prometheus 9090
kubectl port-forward -n istio-system svc/grafana 3000
kubectl port-forward -n istio-system svc/tracing 16686:80
```

## License

Apache License 2.0, see [LICENSE](https://github.com/simonpasquier/snowcamp-io-kiali/blob/master/LICENSE).

