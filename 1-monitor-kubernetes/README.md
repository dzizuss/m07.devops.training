# Exercise: Monitor Kubernetes System With Prometheus and Grafana

Design and run a monitoring stack for a single-node k3s lab cluster. By the end of the exercise you will have Prometheus scraping Kubernetes control-plane and node metrics, and Grafana dashboards visualizing cluster health.

## Learning goals

- Deploy Prometheus and Grafana on k3s using upstream Helm charts
- Collect node resource metrics (CPU, memory, filesystem) and core Kubernetes metrics (APIServer, kubelet, scheduler, etc.)
- Validate data flow end-to-end (Prometheus targets → Grafana dashboards)
- Practice common day-2 operations: troubleshooting pods, exposing services securely, and cleaning up

## Preparation

- Define a namespace for your team
- Define what port your team is going to use, choose between 3000 and 9090
- Import the kubeconfig secret in all your jobs

## Exercise

Create a pipeline with the following blocks:

1. **Deploy**: deploy Prometheus and Grafana using Helm

- Create namespace for your chosen team name
- Switch to the namespace
- Add Helm charts:
  - `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
  - `helm repo update`
- Install the chart:

```shell
helm install monitoring prometheus-community/kube-prometheus-stack --values monitoring-config.yaml
```

- Wait for rollout to complete

## Experiment with Prometheus

Run the following commands from your machine (you will need to have kubectl installed and the kubeconfig file in `$HOME/.kube/config`)

First: switch to your teams namespace, then...

If your team uses port 9090

   ```bash
   kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
   ```

If your team uses port 9001

   ```bash
   kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9091
   ```

2. Visit `http://localhost:9090/targets` or `http://localhost:9091/targets` and confirm the following scrape jobs are `UP`:
   - `kubernetes-nodes`
   - `kubelet`
   - `kube-state-metrics`
   - `node-exporter`
3. Query live data (e.g., `node_cpu_seconds_total` or `kubelet_volume_stats_used_bytes`) to confirm metrics are flowing.

## Experiment with Grafana

## Prerequisites

- A running k3s cluster (single node is fine). From the control host you must be able to run `kubectl get nodes`.
- `kubectl` configured against your k3s cluster.
- `helm` v3.9+ available on your PATH.
- Sufficient cluster resources (at least 2 CPU and 3 GiB free memory) for the monitoring stack.

If `helm` is not installed, follow the [official instructions](https://helm.sh/docs/intro/install/) before beginning.

## Exercise steps

### 1. Inspect the cluster

1. Check cluster health: `kubectl get nodes -o wide` and `kubectl get pods -A`.
2. Confirm the `traefik` (default k3s ingress) pod is running; you will use it if you later expose dashboards via HTTP.

### 2. Prepare a `monitoring` namespace

```bash
kubectl create namespace monitoring
```

- If the namespace already exists, reuse it.
- Label the namespace to make it easy to find: `kubectl label namespace monitoring purpose=observability`.

### 3. Install the kube-prometheus-stack chart

1. Add the Helm repository:

   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. Create a custom values file `monitoring-values.yaml` with the minimal overrides below:

   ```yaml
   grafana:
     adminPassword: prom-operator
     service:
       type: ClusterIP
   prometheus:
     prometheusSpec:
       serviceMonitorSelectorNilUsesHelmValues: false
   ```

   - Keep the service type as `ClusterIP` so you can practice port-forwarding or ingress exposure manually.
   - The `serviceMonitorSelectorNilUsesHelmValues` flag ensures bundled ServiceMonitors are picked up even if you add your own later.
3. Install the chart:

   ```bash
   helm install monitoring prometheus-community/kube-prometheus-stack \
     --namespace monitoring \
     --values monitoring-config.yaml
   ```

### 4. Verify workloads

Run:

```bash
kubectl get pods -n monitoring
```

Wait until all pods report `Running` or `Completed`. If any pod stays in `Pending`, describe it (`kubectl describe pod …`) and resolve resource or permissions issues.

### 5. Explore Prometheus targets

1. Port-forward Prometheus to your workstation:

   ```bash
   kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
   ```

2. Visit `http://localhost:9090/targets` and confirm the following scrape jobs are `UP`:
   - `kubernetes-nodes`
   - `kubelet`
   - `kube-state-metrics`
   - `node-exporter`
3. Query live data (e.g., `node_cpu_seconds_total` or `kubelet_volume_stats_used_bytes`) to confirm metrics are flowing.

### 6. Explore Grafana dashboards

1. Port-forward Grafana:

   ```bash
   kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
   ```

2. Log in at `http://localhost:3000` with `admin/prom-operator` (or the credentials you set).
3. Open the `Node Exporter / Nodes` dashboard. Check CPU, memory, filesystem, and network panels for the k3s node.
4. Open a Kubernetes dashboard like `Kubernetes / Compute Resources / Cluster`. Verify pod counts, pod CPU, and memory match `kubectl top` output (`kubectl top nodes`, `kubectl top pods -A`).

### 7. (Optional) Expose Grafana via Ingress

If you want browser access without port-forwarding:

1. Create a minimal ingress resource (for the built-in Traefik) that maps `http://grafana.local` to the Grafana service.
2. Update your workstation `/etc/hosts` to resolve `grafana.local` to the cluster node IP.
3. Validate TLS or basic auth if exposing beyond localhost.

### 8. Add a custom alert (stretch goal)

1. Create a `PrometheusRule` to alert when node CPU exceeds 80% for 5 minutes.
2. Check that the rule appears under **Alerts** in the Prometheus UI.
3. (Optional) Configure Alertmanager via the chart values to fire notifications to a Slack webhook or email.

### 9. Collect evidence

- Take screenshots or record commands showing:
  - Prometheus targets page with all `UP`.
  - Grafana node dashboard with visible metrics.
  - A sample alert firing (if you implemented one).
- Summarize troubleshooting steps if you had to fix issues (missing CRDs, resource limits, permissions, etc.).

### 10. Cleanup

When finished, remove the stack to free resources:

```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

If you created an ingress or any PVCs, delete them explicitly (`kubectl delete ingress grafana -n monitoring`, `kubectl delete pvc --all -n monitoring`).

## What to submit

- `monitoring-values.yaml`
- A short report (bullet list) covering:
  - How you exposed Grafana (port-forward, ingress, or other).
  - Which dashboards you used to inspect node resource consumption.
  - Any alerts configured and how you validated them.
- Optional: screenshots or terminal captures that demonstrate successful monitoring.

## Troubleshooting tips

- Confirm CRDs are installed: `kubectl get crd | grep monitoring.coreos.com`. If missing, the Helm chart install may have failed.
- Use `kubectl logs` on `kube-prometheus-stack-operator` for reconciliation errors.
- If Grafana shows “No data”, check Prometheus target status and ensure the `prometheus-kube-state-metrics` pod is healthy.
- k3s runs components as static pods under `/var/lib/rancher/k3s/agent/pod-manifests`; make sure node exporter has permission to read `/proc` and `/sys`.

Happy monitoring!
