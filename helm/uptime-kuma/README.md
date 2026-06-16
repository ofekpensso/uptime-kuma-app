# Uptime Kuma Helm Chart

This Helm chart deploys Uptime Kuma to Kubernetes.

The chart is part of the `uptime-kuma-app` repository and is designed to deploy the Docker image produced by the CI pipeline and pushed to Amazon ECR.

## Current Scope

This chart currently deploys:

* A dedicated Kubernetes namespace, created during install with `--create-namespace`
* A `ServiceAccount` for the application
* A `StatefulSet` for Uptime Kuma
* A persistent volume for `/app/data`
* A regular `ClusterIP` service for application access
* A headless service for StatefulSet identity
* Optional Ingress support, disabled by default

The chart is currently intended for a development/demo environment running on EKS.

## Architecture

Current deployment flow:

```text
GitHub Actions
  -> Docker build
  -> Amazon ECR
  -> Helm chart
  -> EKS
  -> StatefulSet
  -> PVC / EBS
  -> ClusterIP Service
  -> kubectl port-forward
```

The application is initially exposed only inside the cluster using a `ClusterIP` service. External access through AWS Load Balancer Controller, ALB, Route53 and TLS will be added later.

## Why StatefulSet?

Uptime Kuma stores application data under:

```text
/app/data
```

In this project, Uptime Kuma runs with SQLite and a persistent volume. Because the application has persistent state, the chart uses a `StatefulSet` instead of a `Deployment`.

The default replica count is:

```yaml
replicaCount: 1
```

The chart keeps `replicaCount` configurable, but for SQLite/PVC mode the safe operational default is one replica. Horizontal scaling requires an application-supported shared state or database strategy.

## Chart Structure

```text
helm/uptime-kuma/
├── Chart.yaml
├── README.md
├── values.yaml
├── values-dev.example.yaml
├── .helmignore
└── templates/
    ├── NOTES.txt
    ├── _helpers.tpl
    ├── ingress.yaml
    ├── service-headless.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    └── statefulset.yaml
```

## Values Files

### `values.yaml`

This is the generic default values file for the chart.

It does not contain environment-specific values such as:

* AWS account ID
* ECR repository URL
* Image tag
* Secrets
* Passwords
* Tokens

### `values-dev.example.yaml`

This is a safe example file that can be committed to Git.

To create a local development values file:

```bash
cp helm/uptime-kuma/values-dev.example.yaml helm/uptime-kuma/values-dev.yaml
```

Then update:

```yaml
image:
  repository: <aws-account-id>.dkr.ecr.<aws-region>.amazonaws.com/uptime-kuma
  tag: sha-REPLACE_WITH_IMAGE_TAG
```

### `values-dev.yaml`

This file is local only and should not be committed.

It may contain environment-specific values such as:

* ECR repository URL
* Image tag
* StorageClass used in the current cluster

It must not contain secrets.

## Secrets Policy

This Helm chart does not store secrets in Git.

Do not put the following values in any committed values file:

* AWS access keys
* AWS secret keys
* Database passwords
* API tokens
* Private keys
* Real Kubernetes Secret values

Future secret integration should use AWS Secrets Manager and External Secrets Operator.

## Prerequisites

Before installing the chart, make sure the following exist:

* An EKS cluster
* Worker nodes with permission to pull from Amazon ECR
* The Uptime Kuma Docker image already pushed to ECR
* A valid Kubernetes context pointing to the EKS cluster
* A working StorageClass

Check the current cluster context:

```bash
kubectl config current-context
```

Check available StorageClasses:

```bash
kubectl get storageclass
```

In the current dev environment, the available StorageClass is:

```text
gp2
```

A future improvement is to create a dedicated `gp3` StorageClass through Terraform.

## Validate the Chart

Run Helm lint:

```bash
helm lint helm/uptime-kuma -f helm/uptime-kuma/values-dev.yaml
```

Render the final Kubernetes manifests without installing:

```bash
helm template uptime-kuma helm/uptime-kuma \
  -n uptime-kuma \
  -f helm/uptime-kuma/values-dev.yaml
```

Check rendered resource kinds:

```bash
helm template uptime-kuma helm/uptime-kuma \
  -n uptime-kuma \
  -f helm/uptime-kuma/values-dev.yaml | grep -E "kind:|  name:"
```

## Install

Install or upgrade the release:

```bash
helm upgrade --install uptime-kuma helm/uptime-kuma \
  -n uptime-kuma \
  --create-namespace \
  -f helm/uptime-kuma/values-dev.yaml
```

## Verify the Deployment

Check the Pod:

```bash
kubectl get pods -n uptime-kuma
```

Check the PVC:

```bash
kubectl get pvc -n uptime-kuma
```

Check the Services:

```bash
kubectl get svc -n uptime-kuma
```

Expected resources:

```text
uptime-kuma-0              Running
data-uptime-kuma-0         Bound
uptime-kuma                ClusterIP
uptime-kuma-headless       ClusterIP / None
```

## Access Locally

Use port-forwarding:

```bash
kubectl port-forward -n uptime-kuma svc/uptime-kuma 3001:3001
```

Open:

```text
http://localhost:3001
```

## Uninstall and Cleanup

Uninstall the Helm release:

```bash
helm uninstall uptime-kuma -n uptime-kuma
```

Check if the PVC still exists:

```bash
kubectl get pvc -n uptime-kuma
```

If the PVC still exists and this is a disposable dev environment, delete it manually:

```bash
kubectl delete pvc data-uptime-kuma-0 -n uptime-kuma
```

Check persistent volumes:

```bash
kubectl get pv
```

Delete the namespace after the workload and PVC are removed:

```bash
kubectl delete namespace uptime-kuma
```

Only after Kubernetes resources are cleaned up, destroy the infrastructure with Terraform.

## Important Cleanup Note

The StatefulSet creates a PVC using `volumeClaimTemplates`.

By default, Kubernetes may keep PVCs after deleting a StatefulSet to avoid accidental data loss. In this dev project, cleanup must be handled carefully to avoid leaving orphaned EBS volumes.

A planned improvement is to add configurable StatefulSet PVC retention policy:

```yaml
persistentVolumeClaimRetentionPolicy:
  enabled: true
  whenDeleted: Delete
  whenScaled: Retain
```

For dev, `whenDeleted: Delete` is useful because the environment is temporary.

For production, `Retain` is often safer to prevent accidental data loss.

## Ingress

Ingress support exists in the chart but is disabled by default:

```yaml
ingress:
  enabled: false
```

External access will be added later using:

* AWS Load Balancer Controller
* ALB Ingress
* Route53
* ACM / TLS

## Future Improvements

Planned next steps:

* Add configurable PVC retention policy
* Add a Terraform-managed `gp3` StorageClass
* Add AWS Load Balancer Controller
* Enable Ingress through ALB
* Add External Secrets Operator
* Connect Kubernetes Secrets to AWS Secrets Manager
* Add ArgoCD for GitOps deployment
* Add monitoring and logging with Prometheus, Grafana and Loki
