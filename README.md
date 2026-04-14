# fleet-infra

GitOps homelab infrastructure managed by [Flux CD](https://fluxcd.io).

## Architecture

```mermaid
flowchart LR
  GitRepo[/"GitHub Repository"/]

  subgraph Clusters["Clusters (FluxCD)"]
    C_K3D["k3d-lab"]
    C_PROD["production"]
  end

  subgraph Infra["Infrastructure (Controllers)"]
    I_BASE["base/"]
    I_K3D["k3d-lab/"]
    I_PROD["production/"]
  end

  subgraph Apps["Applications (Workloads)"]
    A_BASE["base/"]
    A_K3D["k3d-lab/"]
    A_PROD["production/"]
  end

  GitRepo -->|"Syncs"| Clusters

  C_K3D ==>|"1. Applies"| I_K3D
  C_K3D ==>|"2. Applies"| A_K3D

  C_PROD ==>|"1. Applies"| I_PROD
  C_PROD ==>|"2. Applies"| A_PROD

  I_K3D -.->|"Kustomize"| I_BASE
  I_PROD -.->|"Kustomize"| I_BASE

  A_K3D -.->|"Kustomize"| A_BASE
  A_PROD -.->|"Kustomize"| A_BASE
```

## Quickstart (Local Dev)

We use `k3d` for local development, disabling default k3s components to let Flux manage the stack.

### 1. Create the Cluster

```bash
k3d cluster create fleet-infra \
  --image rancher/k3s:v1.31.1-k3s1 \
  --k3s-arg "--disable=traefik@server:0" \
  --k3s-arg "--disable=servicelb@server:0" \
  --k3s-arg "--disable=local-storage@server:0" \
  -p "80:80@loadbalancer" \
  -p "443:443@loadbalancer" \
  --agents 2
```

### 2. Inject Secrets Decryption Key

Flux needs your Age key to decrypt SOPS-encrypted secrets. Place your `age.agekey` file at the root of the project and run:

```bash
kubectl create namespace flux-system
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=age.agekey
```

### 3. Bootstrap Flux

Export your GitHub credentials and link the cluster to this repository:

```bash
export GITHUB_USER="<your-github-username>"
export GITHUB_TOKEN="<your-personal-access-token>"

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/k3d-lab \
  --personal
```

### 4. Verify Deployment

Watch the GitOps synchronization progress:

```bash
flux get kustomizations --watch
```

Once all Kustomizations are `Ready: True`, your cluster is fully operational! Check your pods with `kubectl get pods -A`.
