# Grafana Setup and Integration with Prometheus on EKS

This guide covers the complete setup of Grafana and its integration with Prometheus for Kubernetes monitoring on an AWS EKS cluster.

---

## 1. Introduction

Grafana is an open-source analytics and monitoring solution used to visualize time-series data. In this setup, Grafana is used to display Prometheus metrics for your Kubernetes cluster running on Amazon EKS.

---

## 2. Prerequisites

* A running EKS cluster
* Prometheus already installed (via Helm)
* `kubectl` access to the cluster
* Helm 3 installed on your local system

---

## 3. Add Grafana Helm Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## 4. Install Grafana Using Helm

```bash
helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace
```

---

## 5. Access Grafana Dashboard

Check the external LoadBalancer URL:

```bash
kubectl get svc -n monitoring grafana
```

Example output:

```
grafana   LoadBalancer   <EXTERNAL-IP>   80:32060/TCP   5m
```

Login credentials:

```bash
kubectl get secret --namespace monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

* **Username:** `admin`
* **Password:** <output of above command>

Open browser:

```
http://<EXTERNAL-IP>
```

---

## 6. Add Prometheus as Data Source in Grafana

1. Go to **Gear Icon (Settings)** → **Data Sources**
2. Click **Add data source** → Choose **Prometheus**
3. Fill details:

   * **Name:** Prometheus
   * **URL:** `http://prometheus-server.monitoring.svc.cluster.local`
4. Click **Save & Test**

---

## 7. Import Kubernetes Monitoring Dashboards

1. Go to **+ Create** → **Import**
2. Use any of the following Dashboard IDs:

   * `315` – Node Exporter Full
   * `6417` – K8s cluster monitoring
3. Select Prometheus as the data source → Click **Import**

---

## 8. Validate Prometheus Targets

Visit Prometheus UI:

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090
```

In browser: `http://localhost:9090`

* Go to **Status** → **Targets**
* Ensure all targets are UP (Node Exporter, Kube State Metrics, etc.)

---

## 9. Useful Prometheus Queries

### Pods in Pending State:

```promql
count(kube_pod_status_phase{phase="Pending"})
```

### Total Number of Nodes:

```promql
count(kube_node_info)
```

### CPU Usage by Node:

```promql
rate(container_cpu_usage_seconds_total{image!=""}[5m])
```

### Memory Usage:

```promql
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
```

---

## 10. Email Alert Integration Example

In `values.yaml` (Alertmanager config):

```yaml
alertmanager:
  config:
    global:
      resolve_timeout: 1m
    receivers:
      - name: 'gmail-notifications'
        email_configs:
          - to: g.prasanth532@gmail.com
            from: kkeducationblr@gmail.com
            smarthost: smtp.gmail.com:587
            auth_username: kkeducationblr@gmail.com
            auth_identity: kkeducationblr@gmail.com
            auth_password: <app-password>
            send_resolved: true
    route:
      receiver: 'gmail-notifications'
      group_wait: 10s
      group_interval: 2m
      repeat_interval: 2m
```

**Note:** Enable 2-step verification in Gmail and generate an App Password to use for `auth_password`.

---

## 11. Troubleshooting Tips

* If Grafana panels show `N/A`, check Prometheus targets.
* Confirm network policies and security groups allow communication.
* Verify Prometheus URL is reachable from Grafana pod.

---

## 12. Conclusion

With this setup, your EKS cluster is now integrated with Prometheus and Grafana for full visibility into your cluster performance, pod states, node metrics, and real-time alerting.

For further custom dashboards, alerts, or queries, reach out to your DevOps team or refer to the [Grafana Labs documentation](https://grafana.com/docs/).
