# Resilient Kubernetes Application

A production-style Kubernetes resilience and platform engineering project deployed on **Amazon EKS**, using **Helm, Kubernetes StatefulSets, persistent EBS storage, HPA, ALB Ingress, Redis, PostgreSQL, and containerized application workloads**.

The project focuses on more than simply deploying an application. It demonstrates how to **identify, troubleshoot, recover from, and harden common Kubernetes failure scenarios**, particularly around persistent storage, application health checks, autoscaling, and Helm-based deployment.



---

## Project Overview

This project implements a resilient multi-component application on Amazon EKS with:

* Frontend application
* Backend application
* PostgreSQL database
* Redis cache
* Persistent EBS-backed PostgreSQL storage
* Kubernetes Services
* AWS Application Load Balancer (ALB) Ingress
* Horizontal Pod Autoscaler (HPA)
* Helm-based application packaging
* Environment-specific Helm values
* Kubernetes health probes
* PersistentVolume (PV) and PersistentVolumeClaim (PVC) management
* PostgreSQL storage recovery
* Storage retention testing
* AWS EBS volume optimization

The project also includes hands-on troubleshooting of several realistic Kubernetes issues encountered during deployment.

---

## Architecture

```text
                         Internet
                            │
                            ▼
                 AWS Application Load Balancer
                            │
                            ▼
                  Kubernetes Ingress
                 resilient-app-ingress
                     ┌──────┴──────┐
                     │             │
                    /              /api
                     │             │
                     ▼             ▼
              Frontend Service   Backend Service
                     │             │
                     ▼             ▼
               Frontend Pods    Backend Pods
                  2 replicas     2+ replicas
                                    │
                                    │
                               HPA: 2–5
                                    │
                                    ▼
                              Application API


                 ┌─────────────────────────┐
                 │     resilient-app       │
                 │       namespace         │
                 └────────────┬────────────┘
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
          PostgreSQL StatefulSet       Redis Deployment
                 │                         │
                 ▼                         ▼
              PostgreSQL               Redis Service
                 │
                 ▼
                PVC
                 │
                 ▼
                PV
                 │
                 ▼
           AWS EBS Volume
```

---

# Technology Stack

| Technology              | Purpose                                 |
| ----------------------- | --------------------------------------- |
| Amazon EKS              | Managed Kubernetes cluster              |
| Kubernetes              | Container orchestration                 |
| Helm                    | Application packaging and configuration |
| AWS ALB                 | External application ingress            |
| PostgreSQL 15           | Persistent relational database          |
| Redis                   | Application cache                       |
| AWS EBS                 | Persistent block storage                |
| HPA                     | Backend autoscaling                     |
| Docker/container images | Application workloads                   |
| YAML                    | Kubernetes and Helm configuration       |
| kubectl                 | Cluster administration                  |
| AWS CLI                 | EBS and AWS resource management         |

---


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/inx5jy0m1g9hw1c18dyd.png)



# Repository Structure

```text
.
├── charts/
├── docs/
│   └── screenshots/
└── manifests/
    ├── backend-hpa.yaml
    ├── backend.yaml
    ├── cache.yaml
    ├── configmap-secret.yaml
    ├── database.yml
    ├── frontend.yaml
    ├── helm/
    │   ├── resilient-app/
    │   │   ├── Chart.yaml
    │   │   ├── postgres-recovery-pvc.yaml
    │   │   ├── templates/
    │   │   │   ├── backend-hpa.yaml
    │   │   │   ├── backend.yaml
    │   │   │   ├── cache.yaml
    │   │   │   ├── configmap-secret.yaml
    │   │   │   ├── database.yaml
    │   │   │   ├── frontend.yaml
    │   │   │   └── ingress.yaml
    │   │   ├── values-dev.yaml
    │   │   ├── values-staging.yaml
    │   │   ├── values-prod.yaml
    │   │   └── values.yaml
    │   └── values-prod.yaml
    ├── ingress.yaml
    ├── namespace.yml
    └── storageclass.yaml
```

> The repository contains both the original Kubernetes manifests and the Helm-based implementation used to package the application.

---

# Kubernetes Components

## Frontend

The frontend is deployed as a Kubernetes Deployment and exposed internally through:


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/xj5imewi28gj09tm74m0.png)



```text
frontend-service:80
```

The production configuration supports multiple replicas to improve availability.

The application was validated through Kubernetes port forwarding and returned:

```text
Kubernetes Resilience Demo - Frontend Active
```

---


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/tpplps59969kx1y4e8fi.png)



---

## Backend

The backend is deployed as a Kubernetes Deployment and exposed through:

```text
backend-service:8080
```

The backend health endpoint returns:


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/lakx49kttuea10ktni5i.png)



```json
{
  "status": "healthy",
  "database": "connected",
  "redis": "connected"
}
```

The backend is configured with an HPA allowing it to scale between:

```text
Minimum replicas: 2
Maximum replicas: 5
CPU target:       50%
```

---

## PostgreSQL

PostgreSQL runs as a Kubernetes StatefulSet:

```text
statefulset/postgres
```

The database uses persistent storage through:


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/lvps2w10erh1slk6hpsh.png)



```text
StatefulSet
    ↓
PersistentVolumeClaim
    ↓
PersistentVolume
    ↓
AWS EBS
```

The final state was:

```text
postgres-0    1/1 Running
```

PostgreSQL connectivity was verified using:

```bash
kubectl exec -n resilient-app postgres-0 -- \
  pg_isready -U dbadmin -d resilient_db
```

Result:

```text
/var/run/postgresql:5432 - accepting connections
```

---

## Redis

Redis is deployed as a Kubernetes Deployment and exposed internally through:

```text
redis-service:6379
```

The final cluster state showed Redis healthy with no container restarts.

---

# Helm

The application is packaged as a Helm chart:

```text
manifests/helm/resilient-app/
```


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/3vkspx3m01wdj3h0wtk8.png)



The chart contains:

```text
Chart.yaml
values.yaml
values-dev.yaml
values-staging.yaml
values-prod.yaml
templates/
```

Environment-specific values allow the application configuration to be adapted for different environments without changing the Kubernetes templates.

Example:

```bash
helm template resilient-app . -f values-prod.yaml
```

The production configuration was rendered successfully.

---

# Helm Validation

The Helm chart was validated before applying changes to the cluster.

## Render the chart

```bash
helm template resilient-app . -f values-prod.yaml
```

## Render to a file

```bash
helm template resilient-app . -f values-prod.yaml > rendered-prod.yaml
```

## Server-side Kubernetes validation

```bash
kubectl apply --dry-run=server -f rendered-prod.yaml
```

This validates the generated resources against the Kubernetes API server without actually applying them.

This was used throughout the project to catch configuration problems before deployment.

---

# Resilience Problems Identified and Resolved

One of the main goals of this project was to troubleshoot real deployment failures rather than simply deploy a predefined manifest.

Several issues were identified.

---

## 1. Conflicting Kubernetes StorageClass

The Helm chart initially attempted to manage a `gp2` StorageClass that already existed in the EKS cluster.

The cluster already contained:

```text
gp2 (default)
provisioner: kubernetes.io/aws-ebs
type: gp2
fsType: ext4
```

The Helm-generated StorageClass attempted to define conflicting parameters.

Kubernetes rejected the update because StorageClass parameters are immutable.

The error was:

```text
The StorageClass "gp2" is invalid:
parameters: Forbidden: updates to parameters are forbidden.
```

### Resolution

The application chart was changed so that it does not attempt to recreate or modify the cluster-level StorageClass.

The application should consume the existing StorageClass rather than treating it as an application-owned resource.

This follows a cleaner separation of responsibilities:

```text
Cluster infrastructure
        │
        └── StorageClass
                │
                ▼
Application Helm chart
        │
        └──    PVC
                │
                ▼
               EBS
```

---

# 2. PostgreSQL Persistent Volume Recovery

The most significant resilience exercise involved PostgreSQL persistent storage.

Initially, Kubernetes reported no PVC/PV resources even though the PostgreSQL pod was still running with an attached EBS-backed filesystem.

Investigation showed that PostgreSQL was using an EBS volume containing valid PostgreSQL data.

The underlying volume was identified using the Kubernetes CSI volume information.

The EBS volume contained the PostgreSQL data directory, including:

```text
PG_VERSION
base/
global/
pg_wal/
pg_xact/
pg_multixact/
pg_logical/
postgresql.conf
pg_hba.conf
```

This confirmed that the database data itself had survived.

---

# 3. Manual PV Recovery

The original EBS volume was associated with:

```text
vol-0ed6eca1c06ba8527
```

A PersistentVolume was recreated to reference the existing EBS volume through the AWS EBS CSI driver.

The recovery PV was configured with:

```yaml
persistentVolumeReclaimPolicy: Retain
storageClassName: gp2
volumeMode: Filesystem
```

The existing EBS volume was referenced through:

```yaml
csi:
  driver: ebs.csi.aws.com
```

The recovery PVC was then explicitly bound to the recovered PV.

After recovery:

```text
PVC → Bound
PV  → Bound
PostgreSQL → Running
```

The PostgreSQL data directory was verified after the volume was reattached.

---

# 4. Persistent Storage Lifecycle Test

A deliberate storage lifecycle test was performed to verify the behavior of the `Retain` policy.

The PostgreSQL StatefulSet was scaled down:

```bash
kubectl scale statefulset postgres --replicas=0 -n resilient-app
```

The PVC was deleted.

The retained PV entered the `Released` state rather than immediately destroying the underlying volume.

The original EBS volume remained available.

A new dynamically provisioned volume was observed during the lifecycle test, demonstrating the distinction between:

* Kubernetes PVC lifecycle
* PV lifecycle
* dynamic provisioning
* retained storage
* underlying AWS EBS lifecycle

The original retained PV was then recovered by removing its stale claim reference and binding a recovery PVC to it.

PostgreSQL was started again and the original database files were confirmed to be present.

This demonstrated that the data could be recovered independently of the original PVC object.

---

# 5. PostgreSQL Storage Optimization

After confirming the database data was recovered and PostgreSQL was healthy, the underlying AWS EBS volume was upgraded from:

```text
gp2
```

to:

```text
gp3
```

The EBS modification was performed using the AWS CLI:

```bash
aws ec2 modify-volume \
  --volume-id <VOLUME_ID> \
  --volume-type gp3 \
  --region us-east-2
```

The volume modification reached:

```text
Progress: 100
State: completed
Target type: gp3
Target IOPS: 3000
Target throughput: 125
```

PostgreSQL remained operational throughout the storage optimization.

### Important Kubernetes detail

The underlying AWS EBS volume is now `gp3`, while the Kubernetes PV retains its existing:

```text
storageClassName: gp2
```

This is intentional in the recovery context.

The Kubernetes StorageClass reference is metadata on the PV and is separate from the actual EBS volume type.

A future production implementation could introduce a dedicated CSI-based `gp3` StorageClass and perform a controlled storage migration rather than changing the bound PV metadata directly.

---

# 6. Backend Container Failure

The backend initially entered:

```text
CrashLoopBackOff
```

The container runtime reported:

```text
exec: "-listen=:8080": executable file not found in $PATH
```

Investigation showed that the Helm chart was using:

```text
httpd
```

while passing arguments intended for:

```text
hashicorp/http-echo
```

The application manifest expected:

```yaml
image: hashicorp/http-echo:latest
```

The Helm values were corrected accordingly.

After the change, the backend pods became healthy:

```text
1/1 Running
```

and the Deployment successfully reached its desired replica count.

---

# 7. PostgreSQL Health Probe Failure

PostgreSQL initially generated:

```text
FATAL: database "dbadmin" does not exist
```

The investigation revealed an important distinction:

```text
dbadmin
```

was the PostgreSQL **role/user**, while:

```text
resilient_db
```

was the actual application database.

The original probes used:

```bash
pg_isready -U dbadmin
```

PostgreSQL interpreted the connection target as the database named after the user:

```text
dbadmin
```

which did not exist.

The probes were corrected to:

```yaml
command:
  - pg_isready
  - -U
  - dbadmin
  - -d
  - resilient_db
```

Both liveness and readiness probes were updated.

After deployment, PostgreSQL logs showed:

```text
database system is ready to accept connections
```

and:

```bash
pg_isready -U dbadmin -d resilient_db
```

returned:

```text
/var/run/postgresql:5432 - accepting connections
```

---

# Horizontal Pod Autoscaling

The backend uses a Horizontal Pod Autoscaler.

Configuration:

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/6xo5wgsnakhigh7yyoz3.png)


```text
Minimum replicas: 2
Maximum replicas: 5
CPU target:       50%
```

The HPA was verified in the cluster:

```bash
kubectl get hpa -n resilient-app
```

Final observed state:


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/ryijlg2mftnscjfsqn4q.png)



```text
TARGETS      MINPODS   MAXPODS   REPLICAS
cpu: 2%/50%  2         5         2
```

The HPA behavior was also observed through Kubernetes events.

During testing, the backend scaled:

```text
2 → 3 replicas
```

and later automatically scaled back:

```text
3 → 2 replicas
```

with Kubernetes reporting:

```text
reason: All metrics below target
```

This confirms that the HPA was actively managing the backend workload rather than merely existing as an unused resource.

---

# Application Networking

The application uses Kubernetes Services for internal communication.

```text
backend-service
    ClusterIP
    port 8080

frontend-service
    ClusterIP
    port 80

postgres-service
    Headless Service
    port 5432

redis-service
    Headless Service
    port 6379
```

The PostgreSQL and Redis services use headless service configurations to support stateful/internal service discovery.

---

# AWS Application Load Balancer

The application is exposed through a Kubernetes Ingress using the AWS Load Balancer Controller.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/4u4434ef4jpbixk1s3dg.png)



Ingress:

```text
resilient-app-ingress
```

Class:

```text
alb
```

The EKS cluster successfully provisioned an AWS Application Load Balancer.

The final cluster state showed an ALB DNS address associated with the Ingress.

---

# Validation and Testing

The final cluster was validated using several Kubernetes commands.

## Pod health

```bash
kubectl get pods -n resilient-app
```

Final observed state:

```text
backend      Running
frontend     Running
postgres     Running
redis        Running
```

All observed pods had:

```text
RESTARTS: 0
```

---

## Deployment health

```bash
kubectl get statefulset,deployment -n resilient-app
```


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/l4ip27m7yvbom56vbx3d.png)



Final state:

```text
postgres    1/1
backend     2/2
frontend    2/2
redis       1/1
```

---

## Persistent storage

```bash
kubectl get pvc,pv -n resilient-app
```

Final PostgreSQL storage:

```text
PVC:           Bound
PV:            Bound
Access Mode:   RWO
Capacity:      1Gi
Reclaim Policy: Retain
```

---

## Services and Ingress

```bash
kubectl get svc,ingress -n resilient-app
```

Verified:

```text
backend-service
frontend-service
postgres-service
redis-service
resilient-app-ingress
```

---

## HPA

```bash
kubectl get hpa -n resilient-app
```

Verified:

```text
MINPODS: 2
MAXPODS: 5
CPU TARGET: 50%
```

---

## PostgreSQL health

```bash
kubectl exec -n resilient-app postgres-0 -- \
  pg_isready -U dbadmin -d resilient_db
```

Result:

```text
/var/run/postgresql:5432 - accepting connections
```

---

## Frontend application test

The frontend service was tested using:

```bash
kubectl port-forward svc/frontend-service 8091:80 -n resilient-app
```

The application returned:

```text
Kubernetes Resilience Demo - Frontend Active
```

---

## Backend application test

The backend service was tested using:

```bash
kubectl port-forward svc/backend-service 8092:8080 -n resilient-app
```

The service successfully accepted connections through the port-forward.

---

# Final Cluster State

The final observed application state was:

```text
Namespace: resilient-app

Backend:
  2/2 Running

Frontend:
  2/2 Running

PostgreSQL:
  1/1 Running

Redis:
  1/1 Running

PostgreSQL PVC:
  Bound

PostgreSQL PV:
  Bound

PV Reclaim Policy:
  Retain

Backend HPA:
  2–5 replicas
  50% CPU target

Ingress:
  AWS ALB

Pod Restarts:
  0
```

---

# Key Resilience Principles Demonstrated

This project demonstrates several important Kubernetes and platform engineering principles.

### 1. Stateful workloads require deliberate storage management

A PostgreSQL pod can be recreated, but database data must live independently from the pod lifecycle.

```text
Pod lifecycle ≠ Data lifecycle
```

---

### 2. PVCs, PVs, StorageClasses, and cloud volumes are different layers

The project demonstrates the relationship between:

```text
Application
    ↓
StatefulSet
    ↓
PVC
    ↓
PV
    ↓
CSI Driver
    ↓
AWS EBS
```

Understanding these layers was essential to recovering the database.

---

### 3. Retain policies provide an additional recovery boundary

Using:

```text
persistentVolumeReclaimPolicy: Retain
```

helps prevent automatic destruction of the underlying persistent volume when a claim is removed.

---

### 4. Health checks must test the real application dependency

A syntactically valid probe can still be logically wrong.

The PostgreSQL probe initially checked the wrong database.

Changing:

```text
pg_isready -U dbadmin
```

to:

```text
pg_isready -U dbadmin -d resilient_db
```

aligned the health check with the actual application database.

---

### 5. Autoscaling should be tested, not assumed

The HPA was observed scaling:

```text
2 → 3 → 2
```

based on CPU utilization.

This provided evidence that the autoscaling configuration was functioning.

---

### 6. Helm should consume infrastructure rather than unnecessarily own it

Cluster-level resources such as StorageClasses should be managed deliberately.

The application chart should consume an appropriate existing StorageClass instead of attempting to redefine a cluster-wide resource with the same name.

---

# Security Considerations

Sensitive configuration should **not** be committed to source control.

The repository should use:

* Kubernetes Secrets
* External Secrets where appropriate
* AWS Secrets Manager or Parameter Store for production environments
* Environment-specific configuration
* `.gitignore` for local secrets
* Secret scanning before pushing to GitHub

Example production architecture:

```text
Application
     │
     ▼
Kubernetes Secret
     │
     ▲
External secret management
     │
     ▼
AWS Secrets Manager
```

> Never commit real passwords, AWS credentials, tokens, kubeconfig files, or private keys to this repository.

---

# Operational Lessons

This project provided hands-on experience with:

* Kubernetes troubleshooting
* EKS operations
* StatefulSets
* PersistentVolumes
* PersistentVolumeClaims
* StorageClasses
* AWS EBS
* AWS EBS CSI
* Kubernetes scheduling
* Kubernetes events
* Health probes
* Container entrypoints
* Helm templating
* Environment-specific values
* Kubernetes Services
* AWS ALB Ingress
* HPA
* Storage recovery
* Data persistence
* Failure investigation
* Production-style validation

---

# Future Improvements

This project is intentionally structured as a foundation for a larger Platform Engineering capstone.

Planned improvements include:

### Infrastructure as Code

* Terraform for EKS
* VPC
* IAM
* Security Groups
* EBS CSI
* AWS Load Balancer Controller
* Supporting AWS infrastructure

### CI/CD

* GitHub Actions
* Helm linting
* YAML validation
* Security scanning
* Container image scanning
* Automated deployment

### GitOps

* Argo CD
* Declarative application delivery
* Environment promotion
* Automated drift detection

### Observability

* Prometheus
* Grafana
* Alertmanager
* Application metrics
* Kubernetes metrics
* SLO/SLI monitoring

### Security

* NetworkPolicies
* Pod Security Standards
* Non-root containers
* Read-only root filesystems where applicable
* RBAC hardening
* Image scanning
* Secrets management
* AWS IAM least privilege

### Reliability

* PodDisruptionBudgets
* Pod anti-affinity
* Multi-AZ scheduling
* Automated backup and restore
* PostgreSQL backup strategy
* Disaster recovery testing
* Recovery Time Objective (RTO)
* Recovery Point Objective (RPO)

### Platform Engineering

The long-term goal is to evolve this project into a reusable internal platform that allows application teams to deploy workloads through standardized:

```text
Infrastructure
    ↓
Kubernetes
    ↓
Helm
    ↓
CI/CD
    ↓
GitOps
    ↓
Observability
    ↓
Security
    ↓
Reliability
```

---

# Useful Commands

## Helm

```bash
helm lint .
```

```bash
helm template resilient-app . -f values-prod.yaml
```

```bash
helm template resilient-app . -f values-prod.yaml > rendered-prod.yaml
```

---

## Kubernetes validation

```bash
kubectl apply --dry-run=server -f rendered-prod.yaml
```

```bash
kubectl get pods -n resilient-app
```

```bash
kubectl get svc -n resilient-app
```

```bash
kubectl get pvc,pv -n resilient-app
```

```bash
kubectl get statefulset,deployment -n resilient-app
```

```bash
kubectl get hpa -n resilient-app
```

```bash
kubectl get ingress -n resilient-app
```

```bash
kubectl get events -n resilient-app --sort-by='.lastTimestamp'
```

---

## PostgreSQL

```bash
kubectl exec -n resilient-app postgres-0 -- \
  pg_isready -U dbadmin -d resilient_db
```

---

## Service testing

Frontend:

```bash
kubectl port-forward svc/frontend-service 8091:80 -n resilient-app
```

Backend:

```bash
kubectl port-forward svc/backend-service 8092:8080 -n resilient-app
```

---

# Project Outcome

The project successfully demonstrated a resilient Kubernetes application running on Amazon EKS.

The final implementation successfully achieved:

* Helm-based application packaging
* Multi-component Kubernetes deployment
* Persistent PostgreSQL storage
* PostgreSQL PV/PVC recovery
* Persistent volume retention testing
* AWS EBS volume recovery
* AWS EBS gp2 → gp3 optimization
* Correct PostgreSQL health probes
* Backend container failure resolution
* Kubernetes Service validation
* AWS ALB Ingress provisioning
* Horizontal Pod Autoscaling
* Application health validation
* Zero pod restarts in the final observed state

Most importantly, the project demonstrated the ability to **troubleshoot and recover a stateful Kubernetes workload instead of simply redeploying it and losing the underlying data**.

---

# Author

**Jamiu Bakre**
Cloud & DevOps Engineer | Kubernetes | AWS | Terraform | CI/CD

Focus areas:

* Cloud Engineering
* DevOps
* Kubernetes
* Platform Engineering
* Infrastructure as Code
* AWS
* CI/CD
* Reliability Engineering

---

## Status

**Assignment status: Completed — Design & Deploy a Resilient Kubernetes Application**

This repository represents the first stage of a broader Platform Engineering journey and is intended to evolve into a more complete production-style platform with Infrastructure as Code, GitOps, observability, security automation, and automated reliability testing.
