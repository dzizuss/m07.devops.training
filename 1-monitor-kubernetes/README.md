# Exercise: Monitor Kubernetes System With Prometheus and Grafana

Design and run a monitoring stack for a single-node k3s lab cluster. By the end of the exercise you will have Prometheus scraping Kubernetes control-plane and node metrics, and Grafana dashboards visualizing cluster health.

## Learning goalsd

- Deploy Prometheus and Grafana on k3s using upstream Helm charts
- Collect node resource metrics (CPU, memory, filesystem) and core Kubernetes metrics (APIServer, kubelet, scheduler, etc.)
- Validate data flow end-to-end (Prometheus targets → Grafana dashboards)
- Practice common day-2 operations: troubleshooting pods, exposing services securely, and cleaning up

## Preparation

- Define a namespace for your team
- Define what port your team is going to use, choose between:
  - For Prometheus: 9090 and 9001
  - For Grafana: 3000 and 3001

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

Run the following commands from your machine (you will need to have kubectl installed and the kubeconfig file in `$HOME/.kube/config`)

First: switch to your teams namespace, then...

If your team uses port 3000

```shell
export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring" -oname)
kubectl port-forward $POD_NAME 3000
```

If your team uses port 3001

```shell
export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring" -oname)
kubectl port-forward $POD_NAME 3001
```

Visit `http://localhost:3000/` or `http://localhost:3001/`  and login with admin admin.

Select "Dashboards" from the left menu -> "Kubernetes / Compute Resources / Cluster". You should see the CPU/MEM utilization for the cluster

## Cleanup

Add a promotion to your pipeline to cleanup, with the following blocks:

1. **Destroy**: destroy  Prometheus and Grafana using Helm

- Switch to the namespace
- Uninstall the chart:

```shell
helm uninstall prometheus-community/kube-prometheus-stack
```

Run the promotion to clean up the cluster.
