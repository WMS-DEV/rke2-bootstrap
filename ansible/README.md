# Ansible — Architecture

## Directory structure

```
ansible/
  playbooks/
    rke2-full-install.yaml     # Entry point: full cluster bootstrap + ArgoCD
    rke2-join-install.yaml     # Entry point: join nodes + ArgoCD enroll
    rke2-scale.yaml            # Entry point: scale cluster nodes up
    rke2-decommission.yaml     # Entry point: decommission cluster nodes
    roles/
      rke2-common/             # Node prep — runs on all hosts
      rke2-master/             # Control plane init and join
      rke2-worker/             # Worker join
      bootstrap-artifacts/     # Fetch kubeconfig + token from first master
      argocd/                  # ArgoCD install, enroll, cutover
      node-drain/              # Cordon and drain a node
      node-uncordon/           # Uncordon a node
      node-remove/             # Full node removal (drain + etcd + uninstall + delete)
    group_vars/
      all.yaml                 # Shared variables (vault, network, versions)
  collections/
    requirements.yml           # ansible.posix, community.general,
                               # kubernetes.core, community.hashi_vault
  ansible.cfg
```

---

## Roles

### `rke2-common`

Runs on all hosts. Validates inventory and controller prerequisites, prepares hardware (sysctl, kernel modules, NetworkManager, swap).

| tasks_from | Description |
|---|---|
| *(default)* | `preflight-localhost` → `preflight-hosts` → `hw_prep` |

**Handlers:** `Apply rke2 sysctl settings`, `Reload NetworkManager`

---

### `rke2-master`

Installs and configures RKE2 server on control-plane nodes.

| tasks_from | Description |
|---|---|
| `init` | Bootstraps the first master (config, cilium manifest, kube-vip, rke2-server) |
| *(default)* | `join-precheck` → `join` — joins additional masters to an existing cluster |

---

### `rke2-worker`

Installs and configures RKE2 agent on worker nodes.

| tasks_from | Description |
|---|---|
| *(default)* | `join-precheck` → `join` |

---

### `bootstrap-artifacts`

Runs on `kube-masters[0]`. Fetches kubeconfig and node token from the first master to local artifact storage. Required before any join operation.

| tasks_from | Description |
|---|---|
| *(default)* | `precheck` → `fetch` → `postcheck` |

Artifacts are written to `bootstrap_artifact_dir` (set in the cluster's `group_vars/all.yaml`).

---

### `argocd`

Runs on localhost. Manages the full ArgoCD lifecycle for a cluster.

| tasks_from | Description |
|---|---|
| *(default / `install`)* | `preflight` → `vault-reconfig` → install stub ArgoCD → postcheck |
| `enroll` | Enroll cluster in a parent ArgoCD (requires `argocd_parent_kubeconfig`) |
| `cutover` | Swap stub ArgoCD for production, apply ApplicationSets root, flip bootstrap label |

---

### `node-drain`

Cordons and drains a node via the Kubernetes API (runs on localhost). Reusable for any maintenance that requires evicting workloads before touching a node.

---

### `node-uncordon`

Uncordons a node via the Kubernetes API (runs on localhost). Pair with `node-drain` for rolling operations such as host maintenance.

---

### `node-remove`

Full node removal. Composes `node-drain`, then removes the etcd member (masters only, gated on actual etcd membership), stops and uninstalls RKE2, and deletes the node object from the cluster.

---

## Entry-point playbooks

### `rke2-full-install.yaml`

Full bootstrap of a standalone cluster with ArgoCD. Typical use: `tools` cluster.

```
all             → rke2-common
kube-masters[0] → rke2-master (init) + bootstrap-artifacts
kube-masters[1:]→ rke2-master (join), serial: 1
kube-workers    → rke2-worker
localhost       → argocd (install)
localhost       → argocd (cutover)
```

### `rke2-join-install.yaml`

Bootstrap a cluster and enroll it into a parent ArgoCD. Typical use: `prod` cluster enrolled into `tools`.

```
all             → rke2-common
kube-masters[0] → rke2-master (init)
kube-masters[0] → bootstrap-artifacts
kube-masters[1:]→ rke2-master (join), serial: 1
kube-workers    → rke2-worker
localhost       → argocd (enroll)
```

The `master_init` play can be skipped with `--skip-tags master_init` when adding nodes to an existing cluster.

---

### `rke2-scale.yaml`

Adds new nodes to a running cluster. Detects which hosts already have RKE2 running and only runs on the new ones. Fails if a `kube-masters` host is found running `rke2-agent` or vice versa.

```
all (new only)              → rke2-common
kube-masters:&rke2_existing → bootstrap-artifacts (run_once)
kube-masters:&rke2_new      → rke2-master (join), serial: 1
kube-workers:&rke2_new      → rke2-worker
```

### `rke2-decommission.yaml`

Removes nodes declared in `decommission-kube-masters` and `decommission-kube-workers` inventory groups. Asserts etcd quorum is maintained before proceeding — at most `floor((n-1)/2)` masters can be removed in a single run.

```
localhost                   → quorum preflight assert
decommission-kube-masters   → node-remove, serial: 1
decommission-kube-workers   → node-remove
```

---

## Tags

| Tag | Plays |
|---|---|
| `always` | `rke2-common` (always runs) |
| `preflight` | `rke2-common` |
| `host_prep` | `rke2-common` |
| `master_init` | `rke2-master (init)` |
| `bootstrap_artifacts` | `bootstrap-artifacts` |
| `master_join` | `rke2-master (join)` |
| `workers` | `rke2-worker`, `rke2-decommission` / `rke2-scale` worker plays |
| `argocd` | `argocd (install)` |
| `argocd_cutover` | `argocd (cutover)` |
| `argocd_enroll` | `argocd (enroll)` |
| `bootstrap` | All RKE2 + ArgoCD install plays |
| `control_plane` | All master plays |
| `masters` | `rke2-decommission` master plays |

---

## Variables

### Shared — `group_vars/all.yaml`

Defaults shared across all clusters. Can be overridden by the cluster inventory's own `group_vars/all.yaml`.

| Variable | Description |
|---|---|
| `ansible_user` | SSH user for all hosts |
| `ansible_ssh_private_key_file` | Path to SSH private key |
| `vault_addr` | Vault API address |
| `vault_kv_mount` | KV v2 mount path |
| `vault_github_app_path` | Vault path for GitHub App credentials |
| `vault_argocd_secret_path` | Vault path for ArgoCD `server.secretkey` |
| `argocd_repo_url` | Git URL for the ArgoCD app-of-apps Helm repo |
| `infra_helm_revision` | Branch/tag/SHA to check out |
| `cilium_version` | Cilium version deployed via manifest |
| `kube_vip_version` | kube-vip image version |

### Cluster-specific — `infra-iac/clusters/<cluster>/ansible/group_vars/all.yaml`

Set per cluster in the calling inventory. All are required — no defaults.

| Variable | Description |
|---|---|
| `bootstrap_artifact_dir` | Local path where kubeconfig and token are written during bootstrap |
| `kubeconfig` | Local path to the cluster kubeconfig (derived from `bootstrap_artifact_dir`) |
| `rke2_token_local_path` | Local path to the node token (derived from `bootstrap_artifact_dir`) |
| `infra_helm_path` | Local path where infra-helm is cloned (derived from `bootstrap_artifact_dir`) |
| `rke2_vip` | Virtual IP for the control plane (managed by kube-vip) |
| `rke2_api_fqdn` | FQDN for the API server (used in TLS SAN) |
| `rke2_master_ips` | List of control-plane node IPs (must match `kube-masters` inventory) |
| `rke2_cluster_domain` | Cluster domain (e.g. `prod.k8s.internal.wmsdev.pl`) |
| `rke2_cluster_cidr` | Pod CIDR |
| `rke2_service_cidr` | Service CIDR |
| `vip_interface` | Network interface kube-vip binds the VIP to |
| `vip_prefix` | Prefix length for the VIP interface |
| `argocd_cluster_group` | ArgoCD cluster group label (e.g. `prod`, `tools`) |
| `vault_k8s_auth_path` | Vault Kubernetes auth mount path for this cluster |

### Runtime secrets

Passed at invocation via `-e`, never stored:

| Variable | Required by | Description |
|---|---|---|
| `vault_token` | `argocd` | Vault token with read access to `secret/` |
| `github_pat` | `argocd (install)` | GitHub PAT for cloning infra-helm |
| `argocd_parent_kubeconfig` | `argocd (enroll)` | Path to parent cluster kubeconfig |
