# k8s-platform

GitOps repository for personal Kubernetes infrastructure. Manages ArgoCD configuration, cluster registration, and shared infrastructure services.

## Repository Structure

```
k8s-platform/
├── argocd/                            # ArgoCD control plane
│   ├── root-app.yaml                  # Bootstrap root application (app-of-apps)
│   ├── app-projects/                  # AppProject definitions
│   ├── application-sets/              # ApplicationSet definitions by provider
│   │   └── proxmox/
│   ├── applications/                  # Per-cluster Application manifests
│   │   └── <env>/<cluster>/
│   └── clusters/                      # Cluster registrations + ExternalSecrets
│       └── <env>/<cluster>/
│
├── charts/                            # Helm chart definitions
│
├── values/                            # Helm values per cluster/provider
│   ├── providers/                     # Provider-scoped defaults
│   │   └── proxmox/
│   └── clusters/                      # Cluster-scoped overrides
│       └── <env>/<cluster>/helm/
│
├── Taskfile.yaml
└── README.md
```

## Layout Rationale

| Directory  | Purpose | Hierarchy |
|------------|---------|-----------|
| `argocd/`  | How things get deployed. ArgoCD control plane resources. | env/cluster |
| `charts/`  | What gets deployed. Helm charts wrapping upstream dependencies. | flat |
| `values/`  | How deployments are configured. Values overrides. | provider or env/cluster |

## Path Convention

```
<env>/<provider>-<region>/
```

| Segment  | Values                         |
|----------|--------------------------------|
| env      | `dev`, `prod`                  |
| cluster  | `proxmox-home`                 |

This convention is consistent across `argocd/clusters/`, `argocd/applications/`, and `values/clusters/`.

## Clusters

| Cluster      | Environment | Provider | Region |
|--------------|-------------|----------|--------|
| proxmox-home | prod        | Proxmox  | home   |
| proxmox-home | dev         | Proxmox  | home   |

## Secret Management

Cluster credentials are **not stored in Git**. Each cluster directory under `argocd/clusters/` contains:

- **`cluster-registration.yaml`** -- non-sensitive ArgoCD cluster secret (server URL, labels). The `config` field is left empty.
- **`externalsecret.yaml`** -- an External Secrets Operator resource that pulls the TLS/auth config from GCP Secret Manager and merges it into the cluster secret at runtime.

GCP Secret Manager stores one secret per cluster using the naming convention `argocd-cluster-<env>-<provider>-<region>`.

## Values Layering

Helm values are resolved in order of increasing specificity:

```
charts/<service>/values.yaml                              # Chart defaults
values/providers/<provider>/<service>.yaml                # Provider-scoped
values/clusters/<env>/<cluster>/helm/<service>.yaml       # Cluster-scoped (when needed)
```

Most services only need the first two layers. The cluster-scoped layer is for services that require genuinely unique per-cluster configuration.

## How It Works

1. The **root app** (`argocd/root-app.yaml`) is applied to the management cluster. It creates child Applications that watch this repository and deploy AppProjects, ApplicationSets, Applications, and cluster registrations.
2. **Cluster registrations** under `argocd/clusters/` register remote clusters with ArgoCD. ExternalSecrets inject credentials from GCP Secret Manager.
3. **ApplicationSets** under `argocd/application-sets/` use the cluster generator to deploy services from `charts/` with values from `values/` to every cluster matching a given provider label.
4. **Applications** under `argocd/applications/` deploy specific workloads to specific clusters.

## Prerequisites

- [ArgoCD](https://argo-cd.readthedocs.io/)
- [External Secrets Operator](https://external-secrets.io/) with a ClusterSecretStore configured for GCP Secret Manager
- [Task](https://taskfile.dev/) (go-task)
- [pre-commit](https://pre-commit.com/)

## Tasks

```bash
task lint                                        # Run yamllint
task pre-commit                                  # Run pre-commit hooks
task helm:deps                                   # Update Helm chart dependencies
task helm:template CHART=cert-manager PROVIDER=proxmox  # Template a chart
task validate                                    # Lint + validate all charts
```
