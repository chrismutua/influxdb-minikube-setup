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
3.  Generate an **API Token** with **Write** permissions for the `telegraf` bucket. You will need this for the next step. For Grafana you will also need a token with **Read** access (see the Grafana Configuration section).
4.  InfluxQL (which the Grafana data source uses) addresses a bucket as a *database*. InfluxDB v2 provides that through a **DBRP mapping**. Check whether one exists:

    ```bash
    influx v1 dbrp list --org <YOUR_INFLUXDB_ORG> --token <YOUR_INFLUXDB_TOKEN>
    ```

    If `telegraf` is not listed, create the mapping — take the bucket ID from `influx bucket list`:

    ```bash
    influx v1 dbrp create \
      --bucket-id <YOUR_BUCKET_ID> \
      --db telegraf \
      --rp autogen \
      --default \
      --org <YOUR_INFLUXDB_ORG> \
      --token <YOUR_INFLUXDB_TOKEN>
    ```

### Step 3: Create the Kubernetes Secret

To avoid hardcoding sensitive credentials in the manifest, the InfluxDB token and organization are stored in a Kubernetes Secret and injected into the DaemonSet as environment variables.

Run the following command in your terminal, replacing `<YOUR_INFLUXDB_TOKEN>` and `<YOUR_INFLUXDB_ORG>` with your actual values:

```bash
kubectl create secret generic telegraf-influxdb-secret \
  --from-literal=INFLUXDB_TOKEN='<YOUR_INFLUXDB_TOKEN>' \
  --from-literal=INFLUXDB_ORG='<YOUR_INFLUXDB_ORG>' \
  -n monitoring
```

### Step 4: Apply the Telegraf Manifests

The manifests live in `manifests/`, one Kubernetes object per file. Applying the directory creates:
*   `manifests/namespace.yaml` — the `monitoring` Namespace everything else lives in.
*   `manifests/serviceaccount.yaml` — the `telegraf` ServiceAccount the DaemonSet runs as.
*   `manifests/clusterrole.yaml` — the `telegraf` ClusterRole, granting read access to nodes, pods, services, endpoints and namespaces, plus the Kubelet's `/metrics`, `/stats` and `/stats/summary` URLs.
*   `manifests/clusterrolebinding.yaml` — the `telegraf` ClusterRoleBinding, which grants that ClusterRole to the ServiceAccount.
*   `manifests/configmap.yaml` — the `telegraf-config` ConfigMap containing the Telegraf configuration.
*   `manifests/daemonset.yaml` — the `telegraf` DaemonSet, which runs Telegraf on every node.

```bash
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/
```

*(The Namespace is applied first because `kubectl apply` reads a directory in filename order, which would otherwise place the namespaced objects ahead of the namespace they need.)*

*(Note: the DaemonSet pins `telegraf:1.40.1` with `imagePullPolicy: IfNotPresent`. Bump that version deliberately rather than tracking `latest`, which can silently move the cluster onto a release that changes or removes configuration options.)*

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

### Step 6: Applying Configuration Changes

Telegraf reads its configuration only at start-up, so editing the `ConfigMap` on its own has no effect. After changing `telegraf.conf` in `manifests/configmap.yaml`, re-apply it and restart the DaemonSet:

```bash
kubectl apply -f manifests/configmap.yaml
kubectl rollout restart daemonset/telegraf -n monitoring
kubectl rollout status daemonset/telegraf -n monitoring
```

---

## 📊 Grafana Configuration

The stack stores its metrics in InfluxDB v2, and these instructions use **InfluxQL**. Add the data source from **Connections -> Data Sources -> InfluxDB**, then configure it as follows:

| Setting | Value | Why |
| --- | --- | --- |
| **Query language** | `InfluxQL` | Selects InfluxDB's v1-compatibility `/query` endpoint. |
| **URL** | `http://localhost:8086` | InfluxDB host and port. The access method is **Server**, so Grafana's backend must be able to reach this address. |
| **Custom HTTP header** (under *Advanced HTTP Settings*) | `Authorization` = `Token <YOUR_INFLUXDB_TOKEN>` | The word `Token`, a space, then your API token. This is how InfluxDB v2 authenticates. |
| **Basic auth** | leave **off** | InfluxDB v2 has no usernames or passwords, only tokens. |
| **Database** (under *InfluxDB Details*) | `telegraf` | The bucket, addressed as a database through a DBRP mapping. |
| **User** / **Password** (under *InfluxDB Details*) | leave **empty** | These apply to InfluxDB 1.x only. |
| **HTTP Method** | `POST` | Grafana's default; POST allows larger queries that GET would reject. |
| **Min time interval** | `30s` | Matches the Telegraf `interval`, so each write lands in its own group. |

Leave **Skip TLS Verify**, **TLS Client Auth** and **With CA Cert** off — this is a local test setup talking plain HTTP, so no certificates are needed.

Click **Save & Test**. A working InfluxQL data source reports *"datasource is working. N measurements found."*

*(The token only needs **Read** access to the `telegraf` bucket. The DaemonSet uses a write token, so for least privilege create a separate read-only token for Grafana under **InfluxDB UI -> Data -> API Tokens -> Generate API Token**, restricted to the `telegraf` bucket with **Read**.)*

### 📈 Example Queries (InfluxQL)

These examples are written in **InfluxQL**, so set the data source's **Query Language** to `InfluxQL` (not `Flux`) before using them. They query the `kubernetes_pod_container` measurement collected by Telegraf and filter on two dependent dashboard variables, `$Namespaces` and `$Pods`.

**Variable: `$Namespaces`**

Lists every namespace with running containers. Under **Dashboard settings -> Variables -> New variable**, set **Type** to `Query`, select your InfluxDB data source, and enter:

```sql
SHOW TAG VALUES FROM "kubernetes_pod_container" WITH KEY = "namespace"
```

**Variable: `$Pods`**

Lists the pods in the selected namespaces. Because this query references `$Namespaces`, it re-runs whenever that selection changes, so the drop-down only ever offers pods belonging to the chosen namespaces:

```sql
SHOW TAG VALUES FROM "kubernetes_pod_container" WITH KEY = "pod_name" WHERE "namespace" =~ /^($Namespaces)$/
```

Enable **Multi-value** and **Include All option** on both variables, then expand **Preview of values** in the variable editor to confirm the lists before saving.

**CPU usage (millicores)**

```sql
SELECT mean("cpu_usage_nanocores"::float) / 1000000
FROM "kubernetes_pod_container"
WHERE ("namespace" =~ /^($Namespaces)$/ AND "pod_name" =~ /^($Pods)$/)
AND $timeFilter
GROUP BY time(1m), "pod_name"::tag
```

*   **Unit:** Custom units -> `mCPU` (nanocores divided by 1,000,000).
*   **Alias by:** `$tag_pod_name`

**Memory working set (MiB)**

```sql
SELECT mean("memory_working_set_bytes"::float) / 1048576
FROM "kubernetes_pod_container"
WHERE ("namespace" =~ /^($Namespaces)$/ AND "pod_name" =~ /^($Pods)$/)
AND $timeFilter
GROUP BY time(1m), "pod_name"::tag
```

*   **Unit:** Custom units -> `MiB` (bytes divided by 1,048,576).
*   **Alias by:** `$tag_pod_name`

Both panels match the tags with `=~` and a regular expression because the variables are multi-value: Grafana interpolates a multi-value selection as a `|`-joined regular expression, so matching with plain `=` would produce a comma-separated string that matches nothing. Keep the parentheses around the variable.

### 🔎 Confirming You Are on InfluxDB v2

InfluxQL works against both InfluxDB 1.x and 2.x, so it is worth confirming which version is running:

```bash
curl -s http://localhost:8086/health
```
A v2 instance reports its version, for example `"version": "v2.9.1"`.

```bash
curl -sI http://localhost:8086/ping
```
v2 answers with `X-Influxdb-Version: v2.x.x` and `X-Influxdb-Build: OSS`.

```bash
influxd version
```
Prints the **server** version. Note that `influx version` prints the *CLI* version instead, which can differ.

**Why InfluxQL works on v2:** Grafana's InfluxQL mode uses the v1-compatibility `/query` endpoint that InfluxDB v2 still provides, addressing the bucket through the **DBRP mapping** created in Step 2.

---

## 🛠️ Troubleshooting Guide (What We Learned)

### 1. `403 Forbidden` on `/stats/summary`
**Cause:** The Minikube Kubelet rejects the Service Account token because it doesn't know how to validate it.
**Fix:**
1.  Restart Minikube with `--extra-config=kubelet.authentication-token-webhook=true`.
2.  Ensure `manifests/clusterrole.yaml` includes `nodes/stats` in the `resources` list.

### 2. `403 Forbidden` on `/pods`
**Cause:** The Telegraf ServiceAccount lacks permission to list pods via the Kubelet's proxy endpoint.
**Fix:** Add `nodes/proxy` to the `resources` list in `manifests/clusterrole.yaml`.

### 3. Telegraf `fieldpass` Deprecation Error
**Cause:** Telegraf v1.40.0 removed the `fieldpass` option in favor of `fieldinclude`.
**Fix:** Replace `fieldpass` with `fieldinclude` in the `telegraf.conf` ConfigMap.
*(The DaemonSet now pins `telegraf:1.40.1`, so an unannounced version bump can no longer reintroduce this class of breakage.)*

### 4. Grafana "No Results" for Kubernetes Metrics
**Cause:** Incorrect data source configuration or querying the wrong time range.
**Fix:**
*   Verify that the Grafana data source is pointing to the correct bucket (`telegraf`).
*   Confirm the data source actually authenticates — **Save & Test** should report *"datasource is working. N measurements found."*
*   Check that the dashboard's time range covers a period when Telegraf was actively collecting and sending data.
*   Temporarily comment out the `fieldinclude` option in the Telegraf ConfigMap to ensure all metrics are flowing without filtering.

---

## 🔐 Security Note

This setup uses a standard Kubernetes Secret for local testing. **Base64 is not encryption.** If you commit a standard Secret manifest to GitHub, your token is exposed.
