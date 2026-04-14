# fleet-infra

This repository contains the Infrastructure as Code and GitOps configuration for the homelab, fully managed by Flux CD.

## Development Setup

To test and develop the infrastructure locally, we use `k3d`. The local cluster is created with the default k3s components disabled (Traefik, ServiceLB, Local Storage) because we manage our own stack (Traefik, Tailscale Operator, and Rancher Local Path Provisioner) via Flux.

### 1. Create the Local Cluster

Run the following command to provision the local `fleet-infra` cluster and map the HTTP/HTTPS ports to your host:

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

### 2. Inject the SOPS Age Key (Secrets Decryption)

Before Flux can successfully deploy the infrastructure, it needs the private Age key to decrypt SOPS-encrypted secrets (such as the Tailscale OAuth credentials).

Assuming you have your `age.agekey` file at the root of the repository, create the `flux-system` namespace and inject the secret:

```bash
kubectl create namespace flux-system
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=age.agekey
```

### 3. Bootstrap Flux CD

Before running the bootstrap command, export your GitHub credentials. This allows Flux to authenticate with the GitHub API and automatically configure the necessary deploy keys for the repository:

```bash
export GITHUB_USER="<your-github-username>"
export GITHUB_TOKEN="<your-personal-access-token>"
```

Now, link the cluster to this Git repository so Flux can start the reconciliation loop. Adjust the repository name and branch if necessary:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/k3d-lab \
  --personal
```

### 4. Verify the Deployment

You can monitor the progress of the GitOps synchronization with:

```bash
flux get kustomizations --watch
```

Once all Kustomizations are `Ready: True`, your base infrastructure is operational. You can verify the Tailscale operator and Homepage deployment:

```bash
kubectl get pods -A
flux get helmreleases -A
```
