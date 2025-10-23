# Module 7 - Monitoring on Kubernetes

**Goal**: Deploy Prometheus and Grafana to an existing k3s cluster and build a basic dashboard backed by a sample metrics app.

## What you deploy

The manifests in `kubernetes/` create:

- `metrics-app`: a small Python service that exposes the `request_processing_seconds` metric on port 8000.
- `prometheus`: collects metrics from itself and from `metrics-app`.
- `grafana`: pre-provisioned with a Prometheus data source.

Everything is installed in the `monitoring` namespace.

## Prerequisites

- A running k3s (or any Kubernetes) cluster with `kubectl` already pointing to it.
- Cluster nodes need outbound internet access so the Python container can install `prometheus_client` at startup.

## Step-by-step

1. Inspect `kubernetes/` to understand how the components are wired together. Notice how ConfigMaps are used to pass both the Prometheus scrape configuration and the Python metrics script into the cluster.
2. Apply the manifests:

   ```bash
   kubectl apply -k kubernetes
   ```

3. Watch for everything to become ready:

   ```bash
   kubectl get pods -n monitoring
   ```

   Wait until all pods show `STATUS` `Running` or `Completed`.

4. Port-forward Prometheus so you can reach it from your laptop:

   ```bash
   kubectl port-forward svc/prometheus -n monitoring 9090:9090
   ```

5. Open http://localhost:9090 to confirm the Prometheus UI is running and that the `metrics_app` target is up (Status ➝ Targets).
6. Reach Grafana via the NodePort service exposed on your node(s): `http://<node-ip>:30040`. On single-node k3s installs running locally, `<node-ip>` is often `127.0.0.1`. Log in with `admin`/`admin`, and create a new dashboard panel. Use the Prometheus data source that is already configured and run a query such as `rate(request_processing_seconds_sum[1m])` to see the synthetic load produced by the metrics app.
7. Once you are done exploring dashboards, clean everything up with:

   ```bash
   kubectl delete -k kubernetes
   ```

## Notes

- The original `docker-compose.yml` is still available if you want to compare the container-based setup with the Kubernetes manifests.
- Adjust `kubernetes/grafana.yaml` if you would rather expose Grafana differently (for example, via Ingress or LoadBalancer).
