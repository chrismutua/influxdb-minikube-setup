# AGENTS.md

Instructions for AI coding agents working in this repository. Read this before
making changes.

## What this repository is

Configuration and documentation for a **local Minikube monitoring stack**:
Telegraf runs as a Kubernetes DaemonSet and ships kubelet metrics to InfluxDB v2
on the host machine; Grafana visualizes them. There is no application code, no
build system, no test suite, and no CI.

Eight files are tracked:

| File | Purpose |
| --- | --- |
| `AGENTS.md` | Instructions for AI coding agents working in this repository (this file). |
| `README.md` | The human-facing setup guide. The primary deliverable. |
| `manifests/namespace.yaml` | The `monitoring` Namespace. |
| `manifests/serviceaccount.yaml` | The `telegraf` ServiceAccount. |
| `manifests/clusterrole.yaml` | The `telegraf` ClusterRole (node/pod read access and the Kubelet stats URLs). |
| `manifests/clusterrolebinding.yaml` | The `telegraf` ClusterRoleBinding that grants the ClusterRole to the ServiceAccount. |
| `manifests/configmap.yaml` | The `telegraf-config` ConfigMap holding `telegraf.conf`. |
| `manifests/daemonset.yaml` | The `telegraf` DaemonSet (image pin, env from the Secret, config mount). |

## Manifests: one object per file

* `manifests/` holds **one Kubernetes API object per file**. Never reintroduce a
  multi-document manifest: no file holds more than one object, and no file starts
  with or contains a `---` separator.
* The `Secret` is never a manifest — it is created imperatively (ground rule 1).
* A new object means a new file, and the tracked-files table above, the README's
  Step 4 inventory, and the verification checks below must be updated in the same
  change.
* `kubectl apply -f manifests/` reads the directory in filename order
  (`clusterrole`, `clusterrolebinding`, `configmap`, `daemonset`, `namespace`,
  `serviceaccount`), so the Namespace is applied last. On a cluster that does not
  have `monitoring` yet, apply `manifests/namespace.yaml` first, as the README
  does.

## Ground rules

1. **Never commit credentials.** The InfluxDB token and org live in a Kubernetes
   Secret created imperatively via `kubectl create secret`. Base64 is not
   encryption.
2. **Do not mutate the live Minikube cluster unless explicitly asked.** Read-only
   inspection (`kubectl get`, `kubectl logs`, `influx query`) is encouraged.
   Applying manifests or restarting the DaemonSet needs explicit approval — an
   approved plan that names the command counts.
3. **Keep edits small and scoped.** This repo is documentation plus a handful of
   small manifests; avoid unrelated reformatting or reordering.
4. **Match the existing README style:** emoji `##` section headings, `###`
   subsections, `*   ` bullets, fenced `bash`/`sql` blocks, `->` for UI paths.
5. There is no `.gitignore`. Clean up any scratch files you create so they do not
   show up as untracked noise.

## Plan first, change only on approval

* When the session is in **plan mode**, the only permitted actions are reads and
  presenting the plan. Do not edit, create, commit, or push anything.
* A conversational "looks good", an answer to a question, or an amendment is
  **not** approval. Fold the feedback into the plan and present it again.
* Treat the session mode as authoritative. If you cannot tell whether plan mode
  is active, assume it is and ask.
* Never push to a remote until the change itself has been approved. The git
  workflow below starts only after that.

## Metric filtering is the sharp edge

`[[inputs.kubernetes]]` uses `fieldinclude`. Telegraf's filter modifiers remove
fields from each metric, and **if every field of a metric is filtered out, the
entire measurement is discarded silently** — no error, no log, just missing data.

Field names are per-measurement. The list below is authoritative for Telegraf
v1.40.x and was verified against
`plugins/inputs/kubernetes/kubernetes.go`:

| Measurement | Fields |
| --- | --- |
| `kubernetes_node` | `cpu_usage_nanocores`, `cpu_usage_core_nanoseconds`, `memory_available_bytes`, `memory_usage_bytes`, `memory_working_set_bytes`, `memory_rss_bytes`, `memory_page_faults`, `memory_major_page_faults`, `network_rx_bytes`, `network_rx_errors`, `network_tx_bytes`, `network_tx_errors`, `fs_available_bytes`, `fs_capacity_bytes`, `fs_used_bytes`, `runtime_image_fs_available_bytes`, `runtime_image_fs_capacity_bytes`, `runtime_image_fs_used_bytes` |
| `kubernetes_pod_container` | `cpu_usage_nanocores`, `cpu_usage_core_nanoseconds`, `memory_usage_bytes`, `memory_working_set_bytes`, `memory_rss_bytes`, `memory_page_faults`, `memory_major_page_faults`, `rootfs_available_bytes`, `rootfs_capacity_bytes`, `rootfs_used_bytes`, `logsfs_available_bytes`, `logsfs_capacity_bytes`, `logsfs_used_bytes` |
| `kubernetes_pod_network` | `rx_bytes`, `rx_errors`, `tx_bytes`, `tx_errors` |
| `kubernetes_pod_volume` | `available_bytes`, `capacity_bytes`, `used_bytes` |
| `kubernetes_system_container` | `cpu_usage_nanocores`, `cpu_usage_core_nanoseconds`, `memory_usage_bytes`, `memory_working_set_bytes`, `memory_rss_bytes`, `memory_page_faults`, `memory_major_page_faults`, `rootfs_available_bytes`, `rootfs_capacity_bytes`, `logsfs_available_bytes`, `logsfs_capacity_bytes` |

Gotchas:

* Node network fields are `network_rx_bytes` / `network_tx_bytes`; pod network
  fields are `rx_bytes` / `tx_bytes`. Confusing the two deletes pod network data.
* `restart_count` and `status_phase` are **not** emitted by this plugin. Pod
  restart counts require kube-state-metrics or the `kube_inventory` input.
* `memory_working_set_bytes` is the deliberate memory signal here; do not swap it
  back to `memory_usage_bytes` without being asked.
* Telegraf does not hot-reload its config. A `ConfigMap` change requires both
  `kubectl apply -f manifests/configmap.yaml` **and**
  `kubectl rollout restart daemonset/telegraf -n monitoring`.

## Audience and scope

The README is written for someone setting up the whole stack from scratch on a
local Minikube host: InfluxDB v2 (organization, bucket, tokens, DBRP mapping),
the Kubernetes Secret, the Telegraf DaemonSet, and the Grafana data source.

* Keep it self-contained. Assume no prior context, no existing InfluxDB data,
  and no knowledge of how this repository evolved.
* Document what a working setup requires — never present a step as optional or
  omitted because of the repository's history.
* Include exact commands and the expected result wherever it helps the reader
  confirm they are on track.
* The data source instructions use **InfluxQL**. If Flux guidance is ever added,
  present it as a clearly-labelled alternative, not a replacement.

## Verifying a change

No test suite exists. Before committing:

```bash
# 1. Every manifest is a single, parseable object
python3 -c "
import glob, yaml
for p in sorted(glob.glob('manifests/*.yaml')):
    docs = [d for d in yaml.safe_load_all(open(p)) if d]
    assert len(docs) == 1, (p, len(docs))
    print(p, docs[0]['kind'], docs[0]['metadata']['name'])
"

# 2. Markdown fences are balanced (expect an even number)
for f in README.md AGENTS.md; do printf '%s: ' "$f"; awk '/^```/{n++} END{print n}' "$f"; done

# 3. The embedded Telegraf config is valid TOML
python3 -c "import yaml,tomllib; d=[x for x in yaml.safe_load_all(open('manifests/configmap.yaml')) if x and x['kind']=='ConfigMap'][0]; tomllib.loads(d['data']['telegraf.conf'])"

# 4. The live cluster would be unchanged by the manifests (read-only, applies nothing)
kubectl apply --dry-run=client -f manifests/

# 5. The diff contains only what you intended
git diff --stat && git diff
```

Any field name added to `fieldinclude` must appear in the table above. When in
doubt, check the plugin source:

```bash
curl -sS https://raw.githubusercontent.com/influxdata/telegraf/v1.40.0/plugins/inputs/kubernetes/kubernetes.go \
  | grep -oE 'fields\["[a-z_]+"\]'
```

## Git workflow: PR-less fast-forward (required)

Every change lands on `main` through a short-lived branch and a **fast-forward
merge**. No pull requests, no merge commits.

```bash
# 1. Start from an up-to-date main and confirm it is level with the remote
git fetch origin
git rev-list --left-right --count main...origin/main   # must print: 0	0

# 2. Branch (kebab-case, descriptive)
git switch -c add-something

# 3. Make the change, verify it, then commit (see conventions below)
git add <files>
git commit -m "Short imperative summary"

# 4. Publish the branch
git push -u origin add-something

# 5. Fast-forward main
git switch main
git merge --ff-only add-something

# 6. Publish main and confirm both ends agree
git push origin main
git status -sb
git rev-list --left-right --count main...origin/main   # must print: 0	0
```

Rules:

* If `git merge --ff-only` refuses, **stop and report**. Do not fall back to a
  merge commit and do not rebase without asking.
* If `main` moved on the remote, run `git pull --ff-only origin main` first, then
  retry.
* Never force-push, never rewrite published history, and never commit directly to
  `main` and push without the branch step.
* Leave the merged branch in place; the maintainer deletes branches.

## Commit messages

* Imperative mood, sentence case:
  `Add InfluxQL CPU and memory query examples to README`.
* No Conventional Commits prefixes (`feat:`, `fix:`), no trailing period.
* Keep the subject under ~72 characters. Bodies are optional and rare — plain
  bullet points if genuinely needed.
* No `Co-Authored-By` or `Signed-off-by` trailers.

## Local environment notes

* `kubectl` context: `minikube` (cluster running). Host tools available:
  `telegraf` (1.40.0), `influx` CLI, `kubectl`.
* InfluxDB v2 on `http://localhost:8086` (bucket `telegraf`); Grafana on
  `http://localhost:3000`.
* In-cluster Telegraf may run a different patch release than the host binary
  (observed: cluster 1.40.1, host 1.40.0).
* The DaemonSet pins `telegraf:1.40.1` with `imagePullPolicy: IfNotPresent`. Bump
  the version deliberately: Telegraf removes deprecated options over time (for
  example `fieldpass` was replaced by `fieldinclude`), so tracking `latest` can
  break a working configuration without any change to this repository.
* Verified versions: InfluxDB v2.9.1 (systemd unit `influxdb.service`), Grafana
  13.2.2, Telegraf 1.40.1 in-cluster.
* The Minikube node may be unable to reach Docker Hub (DNS lookups fail with
  `server misbehaving`), so pulling a *new* image tag fails with `ErrImagePull` /
  `ImagePullBackOff`. If a version bump stalls the rollout, load it locally:
  `minikube image load telegraf:<version>` — or, when that digest is already
  cached under another tag,
  `minikube image tag docker.io/library/telegraf:latest docker.io/library/telegraf:<version>`.
* On this machine plain `git fetch` / `git push` can fail with
  `Bad owner or permissions on /etc/ssh/ssh_config.d/20-systemd-ssh-proxy.conf`.
  The cause is that file being a symlink owned by `nobody:nogroup`, which makes
  SSH reject the whole system config. Work around it per-command, without editing
  any config file:

  ```bash
  GIT_SSH_COMMAND="ssh -F $HOME/.ssh/config" git push origin main
  ```

  The durable fix is a system-level permission fix on that file; do not attempt
  it unprompted.

## Known issues (not yet fixed)

Listed so you do not mistake them for intent. Fix only when asked:

1. README Step 3 creates the Secret in a namespace that Step 4's manifests create.
2. README Troubleshooting §4 still suggests temporarily commenting out
   `fieldinclude`, which removes all filtering and risks high cardinality.
