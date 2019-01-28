This directory demonstrates some of the key Kubernetes features like scheduling, scaling and resiliency.

It uses the [hellopod](https://github.com/simonpasquier/hellopod) application which listens on port 8080 and returns a greeting message including its hostname. The application also has endpoints exposing its healthiness and readiness as well as ways to stop/crash itself.

```bash
$ curl http://localhost:8080
Hello from simon-laptop!
```

## Getting started

First start the application exposed as a service with 3 pods.

```bash
kubectl  apply -n default -f hellopod/
kubectl config set-context --current --namespace default
kubectl get services,deployments,pods
```

Check that the service runs fine.

```bash
kubectl run curl --image radial/busyboxplus:curl -ti --rm sh -c 'while curl http://hellopod.default.svc.cluster.local:8080/; do sleep 1; done'
```

Open another terminal window to watch the pods.

```bash
watch kubectl get pods -n default
```

## Scaling up

Scale the deployment from 3 to 5 pods.

```bash
kubectl scale deployments/hellopod --replicas=5
```

## Liveness

Flag one pod as unhealthy and check in the other window that the pod is restarted by Kubernetes.

```bash
kubectl run curl --image radial/busyboxplus:curl -ti --rm sh -c 'curl -XDELETE http://hellopod.default.svc.cluster.local:8080/healthz'
```

Flag one pod as not ready.

```bash
curl -XDELETE http://$HELLOPOD_IP:8080/ready
```

Send a few requests and check that the unready pod has been evicted.

```bash
kubectl run curl --image radial/busyboxplus:curl -ti --rm sh -c 'for i in $(seq 0 20); do curl http://hellopod.default.svc.cluster.local:8080/; done'
```

Exit gracefully a random pod and check that the pod has been restarted.

```bash
kubectl run curl --image radial/busyboxplus:curl -ti --rm sh -c 'curl http://hellopod.default.svc.cluster.local:8080/quit'
```

Crash a random pod and check that Kubernetes restarts/respawns the pod. Depending on the cluster's history, the pod may go into the CrashLoopBackOff status.

```bash
kubectl run curl --image radial/busyboxplus:curl -ti --rm sh -c 'curl http://hellopod.default.svc.cluster.local:8080/fail'
```

## Cleanup

Delete all the resources.

```bash
kubectl delete deployments,services -l app=hellopod
```
