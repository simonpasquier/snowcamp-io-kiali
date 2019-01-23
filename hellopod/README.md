This directory demonstrates some of the Kubernetes features like healthiness, readiness and scaling.

```bash
# Start an application exposed as a service with 3 pods.
kubectl  apply -n default -f hellopod/

# Check that everything is started.
kubectl get -n default services,deployments,pods

# Get the service's IP address.
export HELLOPOD_IP=$(kubectl  get svc/hellopod -n default -o jsonpath='{ .spec.clusterIP }')

# Send some requests and see that each request hits a random pod.
while curl http://$HELLOPOD_IP:8080/; do sleep 1; done

# Watch pods in a separate terminal window.
watch kubectl get pods -n default

# Flag one pod as unhealthy and check that the pod has been restarted.
curl -XDELETE http://$HELLOPOD_IP:8080/healthz

# Flag one pod as not ready.
curl -XDELETE http://$HELLOPOD_IP:8080/ready

# Send some requests and check that the unready pod has been evicted.
while curl http://$HELLOPOD_IP:8080/; do sleep 1; done

# Exit gracefully a random pod and check that the pod has been restarted.
curl http://$HELLOPOD_IP:8080/quit

# Fail hard a random pod and check that the pod's status is going to Error then CrashLoopBackOff and Ready.
curl http://$HELLOPOD_IP:8080/fail
```
