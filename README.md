# gke-deployment-chart

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/gke-deployment-chart)](https://artifacthub.io/packages/helm/gke-deployment-chart/gke-deployment-chart)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![GKE Only](https://img.shields.io/badge/platform-GKE%20only-blue?logo=google-cloud)

> **GKE-only chart.**
> This chart is designed and tested exclusively for **Google Kubernetes Engine (GKE)**.
> It is built around **GCP Workload Identity** as the primary authentication mechanism — pods
> authenticate to GCP services (Cloud SQL, GCS, Pub/Sub, Secret Manager, etc.) without
> any credential files or Kubernetes Secrets. It is **not intended** for EKS, AKS, or
> self-managed Kubernetes clusters.

A production-ready, security-hardened Helm chart for GKE deployments.

## Features

- **GKE Workload Identity** — bind Kubernetes SA to GCP SA without storing credentials
- **Deployments** with configurable replica count and rolling update strategy
- **Horizontal Pod Autoscaler (HPA)** — CPU-based and external metric scaling
- **Vertical Pod Autoscaler (VPA)** — automatic resource right-sizing
- **Pod Disruption Budget (PDB)** — guaranteed availability during disruptions
- **Network Policy** — fine-grained pod-to-pod traffic control
- **RBAC** — least-privilege service account and role bindings
- **Ingress** — supports GCE (Google Cloud Load Balancer), nginx, and Traefik
- **PersistentVolumeClaim (PVC)** — optional persistent storage
- **ConfigMap & Secret** — env vars, file secrets, and settings mounts
- **CloudSQL Proxy** — sidecar support for Google Cloud SQL via Workload Identity
- **Security hardening** — non-root, read-only filesystem, dropped capabilities, seccomp

## Installing the Chart

Add the Helm repository:

```bash
helm repo add gke-deployment-chart https://rupeshkmr116.github.io/k8s-deployment-chart
helm repo update
```

Install the chart:

```bash
helm install my-release gke-deployment-chart/gke-deployment-chart
```

Install with a custom values file:

```bash
helm install my-release gke-deployment-chart/gke-deployment-chart \
  -n my-namespace \
  --create-namespace \
  -f my-values.yaml
```

## Uninstalling the Chart

```bash
helm uninstall my-release -n my-namespace
```

## Configuration

The following table lists the key configurable parameters and their defaults.

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Container image repository | `nginx` |
| `image.tag` | Container image tag | `""` (uses appVersion) |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `serviceAccount.create` | Create Kubernetes ServiceAccount | `true` |
| `serviceAccount.name` | KSA name (must match IAM binding) | `""` |
| `serviceAccount.annotations` | Annotations — set `iam.gke.io/gcp-service-account` here | `{}` |
| `service.type` | Kubernetes Service type | `ClusterIP` |
| `service.targetPort` | Container port to expose | `80` |
| `ingress.enabled` | Enable Ingress resource | `true` |
| `ingress.className` | Ingress class (`gce`, `nginx`, etc.) | `""` |
| `resources.limits.cpu` | CPU limit | `100m` |
| `resources.limits.memory` | Memory limit | `128Mi` |
| `hpa.autoscaling.enabled` | Enable HPA | `false` |
| `vpa.enabled` | Enable VPA | `false` |
| `pdb.enabled` | Enable PodDisruptionBudget | `false` |
| `networkPolicy.enabled` | Enable NetworkPolicy | `false` |
| `rbac.create` | Create RBAC resources | `true` |
| `pvc.enabled` | Enable PVC | `false` |
| `cloudsql.enabled` | Enable CloudSQL proxy sidecar | `false` |
| `podSecurityContext` | Pod-level security context | see `values.yaml` |
| `securityContext` | Container-level security context | see `values.yaml` |

See [values.yaml](gke-deployment-chart/values.yaml) for all available parameters and their documentation.

## Artifact Hub publishing checklist (owner: `rupesh1050`)

If Artifact Hub cannot validate ownership or ingest updates, verify these exact items:

1. `artifacthub-repo.yml` is committed in the **root of your `gh-pages` branch**.
2. `repositoryID` in `artifacthub-repo.yml` is set to the ID shown in Artifact Hub for your repository.
3. The owner email in `artifacthub-repo.yml` matches the email used by your Artifact Hub account (`rupeshkmr116@gmail.com`).
4. `index.yaml` exists and is reachable at:
   `https://rupeshkmr116.github.io/k8s-deployment-chart/index.yaml`
5. The chart package tarball is reachable from that `index.yaml`.

## GKE Workload Identity

This chart is built around [GKE Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity), the recommended way to give GKE workloads access to GCP APIs without managing service account key files.

### How it works

```
Pod → Kubernetes SA (KSA) ──(annotation binding)──► GCP SA (GSA) → GCP IAM roles
```

The chart creates a Kubernetes ServiceAccount annotated with a GCP Service Account email. GKE's Workload Identity webhook injects credentials so your pod can call any GCP API transparently.

### Prerequisites

1. **Workload Identity enabled on your GKE cluster:**

   ```bash
   # New cluster
   gcloud container clusters create my-cluster \
     --workload-pool=PROJECT_ID.svc.id.goog \
     --region=REGION

   # Existing cluster
   gcloud container clusters update my-cluster \
     --workload-pool=PROJECT_ID.svc.id.goog \
     --region=REGION
   ```

2. **GKE_METADATA enabled on the node pool:**

   ```bash
   gcloud container node-pools update default-pool \
     --cluster=my-cluster \
     --region=REGION \
     --workload-metadata=GKE_METADATA
   ```

### Step-by-step setup

#### 1. Create a GCP Service Account (GSA)

```bash
export PROJECT_ID=my-gcp-project
export GSA_NAME=my-app-sa
export GSA_EMAIL="${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create ${GSA_NAME} \
  --project=${PROJECT_ID} \
  --display-name="My App Service Account"
```

#### 2. Grant IAM roles to the GSA

```bash
# Cloud SQL access
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${GSA_EMAIL}" \
  --role="roles/cloudsql.client"

# GCS bucket access
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:${GSA_EMAIL}" \
  --role="roles/storage.objectAdmin"

# Secret Manager access
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${GSA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"
```

#### 3. Bind the Kubernetes SA (KSA) to the GCP SA (GSA)

This tells GCP IAM to trust tokens issued by the KSA:

```bash
export NAMESPACE=my-namespace
# KSA name defaults to <release-name>-gke-deployment-chart
# Override with serviceAccount.name in values.yaml
export KSA_NAME=my-release-gke-deployment-chart

gcloud iam service-accounts add-iam-policy-binding ${GSA_EMAIL} \
  --project=${PROJECT_ID} \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${NAMESPACE}/${KSA_NAME}]"
```

> **Important:** The `KSA_NAME` must exactly match the name of the Kubernetes ServiceAccount
> created by this chart. It defaults to `<release-name>-gke-deployment-chart`.
> Set `serviceAccount.name` in your values to use a fixed, predictable name.

#### 4. Annotate the Kubernetes SA via Helm values

```yaml
serviceAccount:
  create: true
  name: "my-app-sa"           # must match KSA_NAME used in the IAM binding above
  automount: false            # not needed with Workload Identity
  annotations:
    iam.gke.io/gcp-service-account: "my-app-sa@my-gcp-project.iam.gserviceaccount.com"
```

Deploy:

```bash
helm upgrade --install my-release gke-deployment-chart/gke-deployment-chart \
  -n my-namespace \
  --create-namespace \
  -f values.yaml
```

#### 5. Verify the binding

```bash
# Check the annotation is on the KSA
kubectl get serviceaccount ${KSA_NAME} -n ${NAMESPACE} -o yaml

# Smoke-test authentication from inside a pod
kubectl run -it --rm wi-test \
  --image=google/cloud-sdk:slim \
  --serviceaccount=${KSA_NAME} \
  --namespace=${NAMESPACE} \
  -- gcloud auth print-identity-token
```

A valid identity token confirms Workload Identity is working. If it fails, check:
- Node pool has `--workload-metadata=GKE_METADATA`
- IAM binding member matches exactly: `PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]`
- The KSA annotation email matches the GSA email

### Full GKE values example

```yaml
serviceAccount:
  create: true
  name: "my-app-sa"
  automount: false
  annotations:
    iam.gke.io/gcp-service-account: "my-app-sa@my-gcp-project.iam.gserviceaccount.com"

# CloudSQL via Workload Identity — no credential file needed
cloudsql:
  enabled: true
  instanceConnectionName: "my-gcp-project:us-central1:my-db"
  privateIp:
    enabled: true
  auto-iam-authn:
    enabled: true
  port: 5432

ingress:
  enabled: true
  className: "gce"
  hosts:
    - host: my-app.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
  seccompProfile:
    type: RuntimeDefault

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

---

## Examples

### Basic deployment with GCE Ingress

```yaml
image:
  repository: gcr.io/my-project/my-app
  tag: "1.0.0"

ingress:
  enabled: true
  className: gce
  hosts:
    - host: my-app.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
```

### Enable HPA

```yaml
hpa:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 80
```

### Enable PDB

```yaml
pdb:
  enabled: true
  minAvailable: 1
```

## Maintainers

| Name | GitHub |
|------|--------|
| Rupesh Kumar | [@rupeshkmr116](https://github.com/rupeshkmr116) |

## License

[MIT](LICENSE)
