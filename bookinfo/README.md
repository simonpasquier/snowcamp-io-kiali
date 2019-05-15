This scenario deploys an application composed of 3 microservices. 

Run the commands from the project's base directory.

## Initial deployment

The initial configuration will direct traffic to the `v1` of each microservice.

```bash
kubectl create namespace bookinfo
# Enable auto-injection of the proxy sidecar
kubectl label namespace bookinfo istio-injection=enabled
kubectl apply -f bookinfo/01_initial_setup/
```

For OpenShift, run:

```bash
oc new-project bookinfo
oc adm policy add-scc-to-user anyuid -z default -n bookinfo
oc adm policy add-scc-to-user privileged -z default -n bookinfo
oc label namespace bookinfo istio-injection=enabled
oc expose service istio-ingressgateway --hostname=bookinfo.demo.example.com --port=80 -n istio-system
oc apply -f bookinfo/01_initial_setup/
```

Go to `http://bookinfo.demo.example.com/productpage` to generate some traffic.

Check how the traffic flow on the Kiali graph page. You can also open the Grafana and Jaeger dashboards.

## Traffic routing

This section highlights some of the Istio features that allow to control how inbound traffic is routed.

### Deploy reviews v2

```bash
kubectl apply -f bookinfo/02_reviews_service_v2/deployment-reviews-v2.yaml
```

Go to the graph page and display the unused nodes. The graph should display `reviews v2` but without traffic.

### Redirect a specific user

We're going to redirect to `reviews v2` only for users authenticated as `jason`.

```bash
kubectl apply -f bookinfo/02_reviews_service_v2/istio-destination-rule.yaml
kubectl apply -f bookinfo/02_reviews_service_v2/istio-v2-authenticated-users.yaml
```

Refresh the product page several times and see that nothing changes.

Login as `jason` (the password doesn't matter) and see that the reviews have black stars now.

Check that the Kiali graph is showing some of the traffic going to `reviews v2`.

### Redirect a fraction of the traffic

We're now going to redirect 30% of the traffic to `reviews v2`.

```bash
kubectl apply -f bookinfo/02_reviews_service_v2/istio-split-70-30.yaml
```

Refresh the product page several times and see that you get different renderings for the reviews part.

## Egress traffic

`v2` of the `details` service uses an external API service (Google Books API) to retrieve additional information about the books.

Before starting, we are going to enable the `blocking-by-default` policy for external services (prior to Istio 1.1.x, this was the default setting).

```bash
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
```

### Deploy details v2 with mirroring

```bash
kubectl apply -f bookinfo/03_details_service_v2/deployment-details-v2.yaml
kubectl apply -f bookinfo/03_details_service_v2/istio-destination-rule.yaml
kubectl apply -f bookinfo/03_details_service_v2/istio-details-mirror-v2.yaml
kubectl apply -f bookinfo/03_details_service_v2/kiali-details-v2.yaml
```

Check that the Kiali graph is showing shadowed traffic going to `details v2` but with 100% of failures.

The new version is instrumented for Prometheus and should be picked up automatically by the Prometheus server. We have also defined a custom Kiali dashboard that is available in the `details-v2` workload page.

You can also open to the Grafana UI and import the application's [dashboard](https://raw.githubusercontent.com/simonpasquier/snowcamp-io-kiali/master/bookinfo/03_details_service_v2/details_dashboard.json). You should see there that outgoing requests are generating errors.  

Check the logs of the `details-v2` workload: you should see connection errors to the external API endpoint.

### Enable external traffic

To allow traffic from our service to the external API, we need to create a `ServiceEntry` resource.

```bash
kubectl apply -f bookinfo/03_details_service_v2/istio-egress-google-api.yaml
```

Verify that the errors go away both in the logs and in the metrics dashboards.

Let's now shift all traffic to `details v2`.

```bash
kubectl apply -f bookinfo/03_details_service_v2/istio-details-all-v2.yaml
```

Refresh the Bookinfo page in your browser and see that the details section has changed.

## Fault injection and remediation

Istio can be used to inject/simulate errors between your services and it can implement counter-measures such as retries and time-outs without having to modify your services.

### Delays

Switch all traffic to v2 of the `reviews` service.

```bash
kubectl apply -f bookinfo/04_fault_injection/istio-reviews-all-v2.yaml
```

Then add a 2 seconds delay to half of the requests to the `ratings` service.

```bash
kubectl apply -f bookinfo/04_fault_injection/istio-ratings-v1-inject-delay.yaml
```

Check the latency graph for the `ratings-v1` workload. You can switch between "Reported from Destination" and "Reported from Source" and notice the differences.

### Timeouts

Add a 1 second timeout to all requests to v2 of the reviews service.

```bash
kubectl apply -f bookinfo/05_timeouts/istio-reviews-v1-timeouts.yaml
```

Combined with the delay injected before, we start to see some errors between `productpage` and `ratings`.


### Retries

To alleviate the delays, we can ask Istio/Envoy to retry requests on the behalf of the service.

```bash
kubectl apply -f bookinfo/06_retries/istio-reviews-all-v2-with-retries.yaml
```

Check with the Kiali graph that most of the errors go away.

## Service-to-service authentication

Istio can automatically setup mutual TLS (mTLS) between your services. The Citadel component will take care of certificate rotations while the proxies enforce traffic encryption and authentication.

Turn on mTLS for all services.

```bash
kubectl apply -f bookinfo/07_mtls/
```

## Cleanup

```bash
kubectl delete namespaces bookinfo
```
