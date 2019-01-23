This scenario deploys a very simple application composed of only one service.

Run the commands from the project's base directory.

## Without sidecar injection

```bash
kubectl create namespace httpbin
kubectl apply -n httpbin -f ./httpbin/
```

Check that the pod has 1 container only.

```bash
kubectl get pods -n httpbin
```

Open the Kiali console, go to the Applications page and check that `httpbin` is flagged with "Missing Sidecar".

## With sidecar injection

```bash
# Enable auto-injection of the proxy sidecar
kubectl label namespace httpbin istio-injection=enabled
kubectl delete pods -l app=httpbin -n httpbin
```

Check that the pod has 2 containers.

```bash
kubectl get pods -n httpbin
```

Generate some load.

```bash
loadtest -rate 5 -uri http://httpbin.snowcamp.example.com/headers -uri http://httpbin.snowcamp.example.com/status/200
```

Explore the Kiali graph page and the different dashboards.

Generate some errors.

```bash
for i in $(seq 0 10); do
  curl -i http://httpbin.snowcamp.example.com/status/500
  sleep 1
done
```

Go back to the Kiali graph and see that the errors are detected.

Click on the "Distributed Tracing" tab to open the Jaeger UI and check a trace with error.

## Cleanup

```bash
kubectl delete namespaces httpbin
```
