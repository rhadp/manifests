# RHAS Platform Manifests

Kubernetes manifests for the [Red Hat Automotive Suite](https://github.com/rhadp) (RHAS) platform components, deployed via ArgoCD onto an OpenShift cluster provisioned by the [rhadp/cluster](https://github.com/rhadp/cluster) repository.

This repository is the GitOps source of truth — ArgoCD continuously syncs these manifests to the cluster with self-healing enabled. Push a change here and ArgoCD applies it automatically.

## How it works

The [rhadp/cluster](https://github.com/rhadp/cluster) installer creates an ArgoCD `AppProject` named `rhas` and one ArgoCD `Application` per component. Each application points at a subdirectory in this repository:

| ArgoCD Application | Source path | What it deploys |
|---|---|---|
| `platform-defaults` | `manifests/platform/` | Namespaces, RBAC (ClusterRoles, RoleBindings), DevSpaces operator subscription |
| `auto-devspaces` | `manifests/devspaces/` | Dev Spaces namespace, CheCluster instance, workspace pre-provisioning |
| `auto-jumpstarter` | `manifests/jumpstarter/` | Jumpstarter operator subscription, service account, RBAC, config secret |
| `auto-builder` | `manifests/builder/` | RHAS Builder operator (placeholder — add manifests here) |

All applications use automated sync with self-heal, so manual drift on the cluster is corrected automatically.

## Repository structure

```
manifests/
├── platform/           # Shared platform resources (deployed first)
│   ├── namespaces.yml              # auto-platform, auto-jumpstarter, automotive-dev-operator-system
│   ├── cluster-roles-admin.yml     # Admin ClusterRoles (platform + pipelines)
│   ├── cluster-roles-user.yml      # Read-only ClusterRoles (platform + pipelines)
│   ├── cluster-roles-sa.yml        # Service account ClusterRole (PaC)
│   ├── role-binding-*.yml          # RoleBindings for GitOps, admins, users
│   └── sub-devspaces.yaml          # OLM Subscription: DevSpaces operator
├── devspaces/          # Red Hat Dev Spaces
│   ├── namespaces.yml              # auto-devspaces namespace
│   ├── ns-admin.yml                # admin-devspaces workspace namespace
│   └── che-cluster-instance.yml    # CheCluster CR (storage, limits, timeouts)
├── jumpstarter/        # Jumpstarter (HIL testing)
│   ├── subscription.yaml           # OLM Subscription: Jumpstarter operator
│   ├── service-account.yaml        # jumpstarter-config-sa + RoleBinding
│   ├── secret-config-token.yaml    # Service account token secret
│   ├── role-binding-gitops.yml     # ArgoCD access to Jumpstarter CRDs
│   ├── role-binding-platform.yml   # jumpstarter-users group access
│   └── role-secrets.yml            # Secret/ConfigMap read access
└── builder/            # RHAS Builder (RHIVOS image builds)
    └── .gitkeep                    # Placeholder — add builder manifests here
```

## Customization

Fork this repository to tailor the platform for your environment.

### 1. Fork and configure

```bash
# Fork via GitHub UI or CLI, then:
git clone https://github.com/your-org/manifests.git
cd manifests
```

### 2. Make changes

Common customizations:

- **Dev Spaces resources** — edit `manifests/devspaces/che-cluster-instance.yml` to adjust CPU/memory limits, storage size, or idle timeout
- **Operator versions** — edit the `Subscription` resources in `manifests/platform/sub-devspaces.yaml` or `manifests/jumpstarter/subscription.yaml` to pin different versions or channels
- **RBAC** — modify ClusterRoles in `manifests/platform/` to adjust permissions for platform-admins or platform-users
- **Namespaces** — edit `manifests/platform/namespaces.yml` or `manifests/devspaces/namespaces.yml` to change namespace names (must match the cluster inventory configuration)
- **Builder** — add Kubernetes manifests to `manifests/builder/` (the directory is recursively synced)

### 3. Point the cluster at your fork

In the [rhadp/cluster](https://github.com/rhadp/cluster) repository, edit `inventory/platform.yml`:

```yaml
platform_gitops_repo_url: "https://github.com/your-org/manifests"
platform_gitops_repo_revision: "main"    # or your branch
```

### 4. Deploy

For a new cluster, run `./install.sh` as usual. To update an existing cluster with your manifest changes:

```bash
./platform.sh
```

ArgoCD picks up changes from your fork and syncs them automatically. You can also push directly to your fork's `main` branch and ArgoCD will reconcile within its polling interval.

## RBAC model

The platform defines three user groups, managed via Keycloak:

| Group | Cluster access | Platform access |
|---|---|---|
| `cluster-admins` | Full cluster admin | — |
| `platform-admins` | — | Full CRUD on platform and pipeline resources |
| `platform-users` | — | Read-only on platform and pipeline resources |

Jumpstarter has its own groups:

| Group | Access |
|---|---|
| `jumpstarter-admins` | Inherited from platform-admins |
| `jumpstarter-users` | Full CRUD on Jumpstarter resources (clients, exporters, leases) + read secrets/configmaps |

## Related repositories

- [rhadp/cluster](https://github.com/rhadp/cluster) — Ansible-based cluster provisioning and platform deployment

## Disclaimer

This is not an officially supported Red Hat product.

## License

See [LICENSE](LICENSE) file for details.
