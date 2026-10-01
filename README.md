# WordPress on Red Hat OpenShift

A containerized WordPress application deployed on Red Hat OpenShift with a MySQL backend, persistent storage, health probes, resource management, and horizontal pod autoscaling.

## Architecture

```mermaid
flowchart TB
    Internet((Internet))

    subgraph OpenShift["Red Hat OpenShift"]
        Route["OpenShift Route"]

        WPService["WordPress Service<br/>ClusterIP :8080"]

        subgraph WP["WordPress Workload"]
            WPPod["WordPress Pod(s)<br/>Readiness + Liveness<br/>Requests / Limits"]
            HPA["HPA<br/>1 → 3 replicas"]
            WPPVC[("WordPress PVC<br/>RWX · 2Gi")]
        end

        MySQLService["MySQL Service<br/>ClusterIP :3306"]

        subgraph DB["MySQL Workload"]
            MySQLPod["MySQL Pod<br/>Secret-based credentials"]
            MySQLPVC[("MySQL PVC<br/>RWO · 1Gi")]
        end
    end

    Internet --> Route
    Route --> WPService
    WPService --> WPPod
    WPPod --> WPPVC
    WPPod --> MySQLService
    MySQLService --> MySQLPod
    MySQLPod --> MySQLPVC
    HPA -. scales .-> WPPod

## Technologies

* Red Hat OpenShift
* Kubernetes
* Containers
* WordPress
* MySQL
* NFS Persistent Storage
* PersistentVolumeClaims (PVCs)
* Kubernetes Secrets
* Resource Requests and Limits
* Readiness and Liveness Probes
* Horizontal Pod Autoscaler (HPA)
* OpenShift Routes

## Project Components

### WordPress

* Deployment
* ClusterIP Service
* OpenShift Route
* Readiness Probe
* Liveness Probe
* CPU and memory requests/limits
* Persistent storage using a PVC
* Horizontal Pod Autoscaler

### MySQL

* Deployment
* ClusterIP Service
* Persistent storage using a PVC
* Database credentials managed through a Kubernetes Secret
* CPU and memory requests/limits

## Storage

The application uses the `nfs-storage` StorageClass available in the OpenShift training environment.

### WordPress

* Storage: 2Gi
* Access Mode: ReadWriteMany (RWX)
* Mount Path: `/var/www/html`

RWX allows multiple WordPress replicas to access the shared application files.

### MySQL

* Storage: 1Gi
* Access Mode: ReadWriteOnce (RWO)
* Mount Path: `/var/lib/mysql/data`

MySQL is deployed as a single replica in this project.

## Resource Management

Resource requests and limits are configured for both workloads.

### WordPress

| Resource | Request | Limit |
| -------- | ------: | ----: |
| CPU      |    100m |  500m |
| Memory   |   128Mi | 512Mi |

### MySQL

| Resource | Request | Limit |
| -------- | ------: | ----: |
| CPU      |    250m | 1 CPU |
| Memory   |   256Mi |   1Gi |

## Health Checks

WordPress uses HTTP-based readiness and liveness probes against:

`/wp-login.php`

### Readiness Probe

Determines whether the WordPress application is ready to receive traffic.

### Liveness Probe

Determines whether the WordPress application is still healthy.

## Autoscaling

A Horizontal Pod Autoscaler is configured for the WordPress Deployment.

* Minimum replicas: 1
* Maximum replicas: 3
* CPU target: 70%

The HPA uses CPU utilization relative to the configured CPU requests.

## Service Discovery

MySQL is not exposed externally.

WordPress communicates with MySQL through the Kubernetes/OpenShift Service:

`mysql-db:3306`

This provides stable service discovery even if the MySQL Pod is recreated.

## Security

Database credentials are managed through a Kubernetes Secret.

The real Secret is intentionally excluded from version control.

Create your local Secret from the provided template:

```bash
cp manifests/mysql/secret.example.yaml manifests/mysql/secret.yaml
```

Edit the generated file and replace the placeholder values with your own credentials.

Then apply it:

```bash
oc apply -f manifests/mysql/secret.yaml
```

The real Secret must never be committed to Git.

## Project Structure

```text
wordpress-openshift/
├── README.md
├── .gitignore
├── docs/
└── manifests/
    ├── mysql/
    │   ├── deployment.yaml
    │   ├── pvc.yaml
    │   ├── secret.example.yaml
    │   └── service.yaml
    └── wordpress/
        ├── deployment.yaml
        ├── hpa.yaml
        ├── pvc.yaml
        ├── route.yaml
        └── service.yaml
```

## Deployment

### 1. Create the Project

```bash
oc new-project wordpress
```

### 2. Create the Database Secret

```bash
cp manifests/mysql/secret.example.yaml manifests/mysql/secret.yaml
```

Edit the Secret and apply it:

```bash
oc apply -f manifests/mysql/secret.yaml
```

### 3. Deploy MySQL

```bash
oc apply -f manifests/mysql/pvc.yaml
oc apply -f manifests/mysql/deployment.yaml
oc apply -f manifests/mysql/service.yaml
```

Verify:

```bash
oc get pods
oc get pvc
oc get svc
```

### 4. Deploy WordPress

```bash
oc apply -f manifests/wordpress/pvc.yaml
oc apply -f manifests/wordpress/deployment.yaml
oc apply -f manifests/wordpress/service.yaml
oc apply -f manifests/wordpress/route.yaml
oc apply -f manifests/wordpress/hpa.yaml
```

Verify:

```bash
oc get pods
oc get pvc
oc get svc
oc get route
oc get hpa
```

## Persistence Testing

### MySQL

Delete the MySQL Pod:

```bash
oc delete pod -l app=mysql-db
oc get pods -w
```

The database data remains available through the persistent volume.

### WordPress

Delete the WordPress Pod:

```bash
oc delete pod -l app=wordpress
oc get pods -w
```

The WordPress files remain available through the persistent volume.

## Verification and Troubleshooting

Useful commands:

```bash
oc get pods -o wide
oc get pvc
oc get svc
oc get route
oc get hpa
oc describe deployment wordpress
oc describe deployment mysql-db
oc logs deployment/wordpress
oc logs deployment/mysql-db
```

## Learning Objectives

This project demonstrates practical experience with:

* Kubernetes/OpenShift Deployments
* Services and service discovery
* OpenShift Routes
* Persistent storage and PVCs
* Kubernetes Secrets
* Resource requests and limits
* Readiness and liveness probes
* Horizontal Pod Autoscaling
* Application persistence
* Basic OpenShift troubleshooting

