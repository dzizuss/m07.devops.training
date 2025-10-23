# Module 7 - Monitoring on Kubernetes

**Goal**: Deploy Prometheus and Grafana to an existing k3s cluster and build a basic dashboard backed by a sample metrics app.

## What you deploy

The manifests in `kubernetes/` create:

- `metrics-app`: a small Python service that exposes the `request_processing_seconds` metric on port 8000.
- `prometheus`: collects metrics from itself and from `metrics-app`.
- `grafana`: pre-provisioned with a Prometheus data source.

Everything is installed in the `monitoring` namespace.

## Prerequisites

- Select a namespace for your team
- Select port numbers for your team:
  - Prometheus: 9090 or 9091
  - Grafana: 30040 or 30080

Open `kubernetes/grafana.yml` and change the port number to your teams number. Commit the file.

## Step-by-step

Create a pipeline with the following blocks:

1. Build image: Build a Docker Image and upload it to Docker Hub

1. **Deploy monitoring**

- Create namespace
- Switch to namespace

   ```bash
   kubectl apply -k kubernetes
   ```

- Wait for rollout to complete

## Configure Prometheus

From your machine, run the following commands:

If your team uses port 9090

   ```bash
   kubectl port-forward svc/prometheus -n monitoring 9090:9090
   ```

Open <http://localhost:9090> to confirm the Prometheus UI is running and that the `metrics_app` target is up (Status ➝ Targets).

## Configure Grafana

Reach Grafana via the NodePort service exposed on your node(s): `http://devops.tomfern.com:30040`.
Create a new dashboard panel. Use the Prometheus data source that is already configured and run a query such as `rate(request_processing_seconds_sum[1m])` to see the synthetic load produced by the metrics app.

Once you are done exploring dashboards, clean everything up with:

   ```bash
   kubectl delete -k kubernetes
   ```

## Notes

- The original `docker-compose.yml` is still available if you want to compare the container-based setup with the Kubernetes manifests.
- Adjust `kubernetes/grafana.yaml` if you would rather expose Grafana differently (for example, via Ingress or LoadBalancer).
