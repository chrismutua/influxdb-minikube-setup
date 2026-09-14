# 📊 Local Minikube Monitoring Stack

This repository contains the configuration to monitor a local Minikube cluster using **Telegraf** as the metrics collector, **InfluxDB** as the time-series database, and **Grafana** for visualization.

The architecture is designed to run entirely on a local machine without requiring any cloud resources. The setup is agnostic to the workloads running on the cluster, meaning it will automatically collect metrics for any pods deployed to Minikube.

## 🏗️ Architecture

*   **Telegraf**: Deployed as a Kubernetes `DaemonSet` inside the Minikube cluster. It runs on every node to collect Kubernetes metrics (CPU, memory, network) from the Kubelet API and sends them to InfluxDB.
*   **InfluxDB**: Runs locally on the host machine and stores the metrics in a bucket named `telegraf`.
*   **Grafana**: Connects to InfluxDB to visualize the metrics.

The Telegraf pods communicate with the host machine's InfluxDB using the Minikube internal DNS name `host.minikube.internal`.

---

## 📋 Prerequisites

*   Minikube installed and running.
*   Workloads deployed in the cluster that you want to monitor.
*   InfluxDB v2.x installed and running on the host machine (port `8086`).
*   Grafana installed and running on the host machine.
*   `kubectl` CLI configured to talk to your Minikube cluster.

---

## 🚀 Setup Instructions

### Step 1: Configure Minikube for Kubelet Authentication

By default, the Minikube Kubelet does not trust Kubernetes Service Account tokens. Telegraf needs to authenticate with the Kubelet to collect node and pod metrics. To enable this, Minikube must be started with the Kubelet authentication webhook enabled.

```bash
minikube stop
minikube start --extra-config=kubelet.authentication-token-webhook=true
```
*(Note: Restarting Minikube will restart your workloads, but Kubernetes controllers will automatically reconcile them back to their desired state.)*

### Step 2: Prepare InfluxDB

1.  Ensure InfluxDB is running on your host machine.
2.  Create an organization (e.g., `<YOUR_INFLUXDB_ORG>`) and a bucket named `telegraf`.
3.  Generate an **API Token** with **Write** permissions for the `telegraf` bucket. You will need this for the next step.

### Step 3: Create the Kubernetes Secret

To avoid hardcoding sensitive credentials in the manifest (making it safe to commit to Git), we store the InfluxDB token and organization in a Kubernetes Secret.

Run the following command in your terminal, replacing `<YOUR_INFLUXDB_TOKEN>` and `<YOUR_INFLUXDB_ORG>` with your actual values:

```bash
kubectl create secret generic telegraf-influxdb-secret \
  --from-literal=INFLUXDB_TOKEN='<YOUR_INFLUXDB_TOKEN>' \
  --from-literal=INFLUXDB_ORG='<YOUR_INFLUXDB_ORG>' \
  -n monitoring
```

### Step 4: Apply the Telegraf Manifest

Apply the `telegraf-daemonset.yaml` manifest to your cluster. This manifest creates:
*   A `monitoring` namespace.
*   A `ServiceAccount` and `ClusterRole` with permissions to read node stats and pod metrics.
*   A `ConfigMap` containing the Telegraf configuration.
*   A `DaemonSet` to run Telegraf on every node.

```bash
kubectl apply -f telegraf-daemonset.yaml
```

### Step 5: Verify the Deployment

Check that the Telegraf pods are running:

```bash
kubectl get pods -n monitoring -l app=telegraf
```

Check the logs for any errors:

```bash
kubectl logs -n monitoring -l app=telegraf --tail=30
```

To see exactly what metrics Telegraf is generating, run it in test mode inside the pod (replace `<pod-name>` with your actual pod name):

```bash
kubectl exec -it <pod-name> -n monitoring -- telegraf --config /etc/telegraf/telegraf.conf --test --input-filter kubernetes
```

---

## 📊 Grafana Configuration

### Query Language: Flux vs. InfluxQL

*   **Flux**: The native query language for InfluxDB 2.x. It requires writing code.
*   **InfluxQL**: The older query language. It supports a **Visual Query Builder** in Grafana, which is what you want for the UI dropdowns.

**To use the Visual Builder in Grafana:**
1.  Go to **Connections -> Data Sources** in Grafana.
2.  Select your InfluxDB data source.
3.  Change the **Query Language** from `Flux` to `InfluxQL`.
4.  Fill in the authentication:
    *   **User**: Your InfluxDB username.
    *   **Password**: Paste your **InfluxDB API Token** here (not your user password).
    *   **Database**: `telegraf`.
5.  Click **Save & Test**.

**Important for InfluxDB 2.x:** You must create a DBRP (Database Retention Policy) mapping for InfluxQL to work. Run this command on your host machine:

```bash
influx v1 dbrp create \
  --bucket-id <your-bucket-id> \
  --db telegraf \
  --rp autogen \
  --org <YOUR_INFLUXDB_ORG> \
  --token <your-api-token>
```

---

## 🛠️ Troubleshooting Guide (What We Learned)

### 1. `403 Forbidden` on `/stats/summary`
**Cause:** The Minikube Kubelet rejects the Service Account token because it doesn't know how to validate it.
**Fix:**
1.  Restart Minikube with `--extra-config=kubelet.authentication-token-webhook=true`.
2.  Ensure the `ClusterRole` includes `nodes/stats` in the `resources` list.

### 2. `403 Forbidden` on `/pods`
**Cause:** The Telegraf ServiceAccount lacks permission to list pods via the Kubelet's proxy endpoint.
**Fix:** Add `nodes/proxy` to the `resources` list in the `ClusterRole`.

### 3. Telegraf `fieldpass` Deprecation Error
**Cause:** Telegraf v1.40.0 removed the `fieldpass` option in favor of `fieldinclude`.
**Fix:** Replace `fieldpass` with `fieldinclude` in the `telegraf.conf` ConfigMap.

### 4. Missing `n_users` Metric (Host Machine Monitoring)
**Cause:** Some modern Linux distributions no longer use or populate `/var/run/utmp`. Telegraf's `system` plugin cannot read it.
**Fix:** Use the `exec` plugin with `loginctl` or `who` to count unique logged-in users.
*   *Script:* `loginctl list-sessions --no-legend | awk '{print $3}' | sort -u | wc -l`
*   *Config:* Use `commands = ["/usr/local/bin/count_unique_users.sh"]` with `data_format = "value"` and `data_type = "integer"`.

### 5. Telegraf `exec` Plugin Pipeline Errors
**Cause:** Telegraf does not run commands in a shell. Using pipes (`|`) directly in the `command` string causes `unrecognized option` errors (e.g., `loginctl: unrecognized option '-u'`).
**Fix:** Wrap the command in a shell script or use `bash -c "..."`.

### 6. Grafana "No Results" for Kubernetes Metrics
**Cause:** Overly strict metric filtering or querying the wrong bucket.
**Fix:**
*   Comment out `fieldinclude` temporarily to ensure data flows.
*   Query InfluxDB directly with: `from(bucket: "telegraf") |> range(start: -15m) |> filter(fn: (r) => r._measurement =~ /kubernetes/)`

---

## 🔐 Security Note for GitOps

This setup uses a standard Kubernetes Secret for local testing. **Base64 is not encryption.** If you commit a standard Secret manifest to GitHub, your token is exposed.

For a production GitOps setup, use **Bitnami Sealed Secrets** or **SOPS**. This allows you to commit encrypted secrets to Git, which are decrypted by a controller inside the cluster.
