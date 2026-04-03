# Tekton CI on AKS

> GitHub Push → Auto-trigger Pipeline on Azure Kubernetes Service

---

## 1. Prerequisites

- Azure CLI installed and logged in: `az login`
- `kubectl` installed
- `tkn` CLI installed: `brew install tektoncd-cli`
- A GitHub repository (public)

---

## 2. Create AKS Cluster

```bash
# Create resource group
az group create \
  --name rg-tekton-learn \
  --location japanwest

# Create AKS cluster (2 nodes, minimum spec for learning)
az aks create \
  --resource-group rg-tekton-learn \
  --name aks-tekton-learn \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys \
  --enable-managed-identity \
  --network-plugin azure \
  --no-wait

# Wait until cluster is ready
az aks wait \
  --resource-group rg-tekton-learn \
  --name aks-tekton-learn \
  --created

# Get kubeconfig
az aks get-credentials \
  --resource-group rg-tekton-learn \
  --name aks-tekton-learn

# Verify
kubectl get nodes
```

> **Note:** AKS automatically creates a second resource group
> `MC_rg-tekton-learn_aks-tekton-learn_japanwest` containing the actual VM,
> VNet, NSG, and Load Balancer resources. Do not modify it manually.

---

## 3. Install Tekton

```bash
# Tekton Pipelines
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Tekton Triggers + built-in Interceptors
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod \
  --all -n tekton-pipelines --timeout=300s
```

---

## 4. YAML Files

Apply all files in order:

```bash
kubectl apply -f tekton/
```

### `00-namespace.yaml`

Creates the `ci` namespace where all Tekton resources will live.

### `01-rbac.yaml`

Creates a ServiceAccount (`tekton-triggers-sa`) and grants it the permissions
needed to run the EventListener. Two levels of permissions are required:

- **Role + RoleBinding** (namespace-scoped): allows reading TriggerBindings,
  TriggerTemplates, Interceptors, and creating PipelineRuns within the `ci` namespace.
- **ClusterRole + ClusterRoleBinding** (cluster-scoped): allows listing
  ClusterInterceptors and ClusterTriggerBindings, which are cluster-wide resources
  used by the GitHub Interceptor.

### `02-secret.yaml`

Stores the GitHub Webhook secret token. The EventListener uses this to verify
that incoming requests genuinely come from GitHub (HMAC-SHA256 signature check).
Replace the value with your own random string before applying.

```yaml
stringData:
  secretToken: "replace-with-your-webhook-secret"
```

### `03-pipeline.yaml`

Defines the Pipeline with two Tasks:

- **clone**: uses the `git-clone` Task from Tekton Hub (via `hub` resolver) to
  check out the repository into a shared workspace PVC.
- **read-file**: runs after `clone`, uses an inline `alpine` container to list
  files and print `README.md`.

The `podTemplate.securityContext.fsGroup: 65532` setting ensures the mounted
PVC directory is writable by the git-clone container.

### `04-triggerbinding.yaml`

Extracts fields from the GitHub push webhook payload and maps them to named parameters:

| Parameter | Source |
|-----------|--------|
| `repo-url` | `body.repository.clone_url` |
| `revision` | `body.after` (commit SHA) |
| `repo-name` | `body.repository.name` |

### `05-triggertemplate.yaml`

Receives the parameters from TriggerBinding and instantiates a PipelineRun.
Each push event creates a new PipelineRun with `generateName` so names never
conflict. Also includes:

- `podTemplate.securityContext.fsGroup: 65532` — fixes PVC permission issue
- `volumeClaimTemplate` with `storage: "500Mi"` — temporary workspace for each run

### `06-eventlistener.yaml`

The HTTP endpoint that GitHub calls. Key settings:

- `serviceType: LoadBalancer` — exposes the listener on a public Azure IP, port 8080
- **GitHub Interceptor** — validates the `X-Hub-Signature-256` header against
  the secret in `02-secret.yaml`, and filters to `push` events only
- References TriggerBinding and TriggerTemplate defined above

---

## 5. Open Port 8080 in Azure NSG

AKS creates a LoadBalancer automatically, but the Network Security Group blocks
inbound traffic by default. Add a rule to allow port 8080:

```bash
# Find NSG name in the MC_* resource group
az network nsg list \
  --resource-group MC_rg-tekton-learn_aks-tekton-learn_japanwest \
  --query "[].name" --output tsv

# Allow inbound 8080
az network nsg rule create \
  --resource-group MC_rg-tekton-learn_aks-tekton-learn_japanwest \
  --nsg-name <NSG-NAME> \
  --name allow-tekton-webhook \
  --priority 1010 \
  --protocol Tcp \
  --direction Inbound \
  --source-address-prefixes '*' \
  --destination-port-ranges 8080 \
  --access Allow

# Get the public IP assigned to the EventListener
kubectl get svc -n ci -l eventlistener=github-webhook
# Note the EXTERNAL-IP value
```

---

## 6. Configure GitHub Webhook

In your GitHub repository, go to **Settings → Webhooks → Add webhook**:

| Field | Value |
|-------|-------|
| Payload URL | `http://<EXTERNAL-IP>:8080` |
| Content type | `application/json` |
| Secret | same value as `02-secret.yaml` `secretToken` |
| Events | Just the push event |
| Active | checked |

After saving, GitHub sends a ping request. The EventListener should return
HTTP 200. You can verify in the webhook's **Recent Deliveries** tab.

---

## 7. Verify the Pipeline

Push any commit to your repository, then:

```bash
# Watch PipelineRuns appear automatically
kubectl get pipelinerun -n ci --watch

# Stream logs of the latest run
tkn pipelinerun logs --last -n ci -f
```

A successful run prints the repository file list and `README.md` contents in
the `read-file` step.

```bash
# View in Tekton Dashboard (local port-forward)
kubectl port-forward -n tekton-pipelines svc/tekton-dashboard 9097:9097
# Open http://localhost:9097

# Clean up old PipelineRuns
kubectl delete pipelinerun --all -n ci
```

---

## 8. Clean Up All Resources

When you are done, delete the resource group. This removes the AKS cluster,
all nodes, the Load Balancer, Public IP, VNet, and the `MC_*` resource group automatically.

```bash
az group delete \
  --name rg-tekton-learn \
  --yes \
  --no-wait

# Confirm both resource groups are gone
az group list --query "[?contains(name,'tekton')].name" --output tsv
```

Also remove the kubeconfig entry for the deleted cluster:

```bash
kubectl config delete-context aks-tekton-learn
kubectl config delete-cluster aks-tekton-learn
```

---

## Appendix: Cost Estimate

Region: Japan West. Prices are approximate and billed per hour.

| Resource | Cost |
|----------|------|
| AKS management fee | Free |
| 2x Standard_B2s nodes | ~¥14 / hour |
| Load Balancer | ~¥2 / hour |
| Public IP | ~¥0.5 / hour |
| **Total** | **~¥17 / hour (~¥410 / day)** |

> Delete the resource group immediately after finishing to avoid unnecessary charges.
