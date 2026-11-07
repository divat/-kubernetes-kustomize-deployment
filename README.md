# Product & Order Services – Kubernetes Kustomize Deployment

A practical Kubernetes deployment project demonstrating how to deploy multiple Spring Boot microservices across DEV and QA environments using **Kustomize, Kubernetes Deployments, Services, ConfigMaps, Secrets, PostgreSQL, and F5 NGINX Ingress Controller**.

The project demonstrates environment-specific configuration, service discovery, database connectivity, application health checks, and HTTP routing through Kubernetes Ingress.

---

## 1. Project Overview

This project contains two Spring Boot microservices:

| Application | Container Port | Database |
|---|---:|---|
| Product API | 8084 | productdb_dev / productdb_qa |
| Order Service | 8085 | orderdb_dev / orderdb_qa |

Both applications use a shared PostgreSQL instance running inside Kubernetes.

Each environment has its own namespace and separate databases.

### Architecture

```text
                    External Client
                          |
                          |
                 F5 NGINX Ingress
                 IngressClass: nginx
                          |
              +-----------+-----------+
              |                       |
          dev.local                qa.local
              |                       |
       DEV Namespace            QA Namespace
              |                       |
       +------+-------+         +-----+--------+
       |              |         |              |
   Product API   Order Service  Product API  Order Service
     :8084          :8085         :8084         :8085
       |              |             |             |
       +--------------+-------------+-------------+
                              |
                       PostgreSQL Service
                       postgres-service
                              |
                       PostgreSQL Pod
                         postgres-0
                              |
                   Persistent Volume Claim
```

**Environment database mapping:**

| Environment | Product API | Order Service |
|---|---|---|
| DEV | productdb_dev | orderdb_dev |
| QA | productdb_qa | orderdb_qa |

The existing `productdb` and `orderdb` databases are preserved.

---

## 2. Technology Stack

- Java / Spring Boot microservices
- Docker container images
- Kubernetes
- Kind Kubernetes cluster
- Kustomize
- F5 NGINX Ingress Controller
- PostgreSQL
- Kubernetes ConfigMaps
- Kubernetes Secrets
- Kubernetes Deployments and ClusterIP Services
- Kubernetes health probes
- kubectl

---

## 3. Project Structure

```text
product-order-kustomize/
│
├── services/
│   │
│   ├── product-api/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   │
│   │   └── overlays/
│   │       ├── dev/
│   │       │   ├── configmap.yaml
│   │       │   └── kustomization.yaml
│   │       │
│   │       └── qa/
│   │           ├── configmap.yaml
│   │           └── kustomization.yaml
│   │
│   ├── order-service/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   │
│   │   └── overlays/
│   │       ├── dev/
│   │       │   ├── configmap.yaml
│   │       │   └── kustomization.yaml
│   │       │
│   │       └── qa/
│   │           ├── configmap.yaml
│   │           └── kustomization.yaml
│   │
│   └── postgres/
│       ├── base/
│       └── overlays/
│           ├── dev/
│           └── qa/
│
├── environments/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── ingress.yaml
│   │
│   └── qa/
│       ├── kustomization.yaml
│       └── ingress.yaml
│
└── README.md
```

---

## 4. Kustomize Architecture

The project uses a Base and Overlay approach.

### Base

The base directory contains common Kubernetes resource definitions.

For example:

- Deployment
- Service
- Container image
- Container port
- Common application configuration

### Overlays

Each environment has its own overlay.

DEV and QA can independently configure:

- Database names
- Replica counts
- Environment variables
- Other environment-specific settings

Example:

**DEV Product API ConfigMap**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: product-api-config
data:
  DB_HOST: postgres-service.default.svc.cluster.local
  DB_PORT: "5432"
  DB_NAME: productdb_dev
```

**QA Product API ConfigMap**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: product-api-config
data:
  DB_HOST: postgres-service.default.svc.cluster.local
  DB_PORT: "5432"
  DB_NAME: productdb_qa
```

The Order Service uses the same approach with `orderdb_dev` and `orderdb_qa`.

---

## 5. PostgreSQL Configuration

PostgreSQL runs in the `default` namespace as a StatefulSet.

Existing Kubernetes resources:

| Resource | Name |
|---|---|
| StatefulSet | postgres |
| Pod | postgres-0 |
| Service | postgres-service |
| Persistent Volume Claim | postgres-data-postgres-0 |
| Database port | 5432 |

The applications connect through the PostgreSQL Kubernetes Service:

```text
postgres-service.default.svc.cluster.local:5432
```

Separate databases provide logical data separation between DEV and QA.

### Database creation

Connect to PostgreSQL:

```bash
kubectl exec -it postgres-0 -n default -- sh
```

Connect using the PostgreSQL role:

```bash
psql -U "$POSTGRES_USER" -d postgres
```

Create the environment databases if they do not already exist:

```sql
CREATE DATABASE productdb_dev;
CREATE DATABASE orderdb_dev;

CREATE DATABASE productdb_qa;
CREATE DATABASE orderdb_qa;
```

List databases:

```sql
\l
```

Exit:

```sql
\q
```

The existing PostgreSQL StatefulSet and persistent volume are reused. The application deployment does not create a new PostgreSQL instance.

---

## 6. Kubernetes ConfigMaps and Secrets

### ConfigMaps

ConfigMaps hold non-sensitive application configuration.

Examples:

- DB_HOST
- DB_PORT
- DB_NAME
- PRODUCT_API_BASE_URL
- SERVER_PORT
- Connection pool settings

### Secrets

Database credentials are stored in a Kubernetes Secret named:

```text
postgres-secret
```

The applications retrieve credentials using `secretKeyRef`.

Because Kubernetes Secrets are namespace-scoped, the Secret must exist in each application namespace.

Copy the existing Secret from `default` into DEV:

```bash
kubectl get secret postgres-secret -n default -o json \
| jq 'del(.metadata.namespace, .metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp, .metadata.managedFields)' \
| kubectl apply -n dev -f -
```

Copy it into QA:

```bash
kubectl get secret postgres-secret -n default -o json \
| jq 'del(.metadata.namespace, .metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp, .metadata.managedFields)' \
| kubectl apply -n qa -f -
```

Verify:

```bash
kubectl get secret postgres-secret -n dev
kubectl get secret postgres-secret -n qa
```

**Security:** Never commit actual Secret manifests, passwords, or credentials to Git.

---

## 7. Kubernetes Health Probes

The Order Service uses Spring Boot Actuator health endpoints.

Liveness:

```text
/actuator/health/liveness
```

Readiness:

```text
/actuator/health/readiness
```

A startup probe is configured to accommodate the application's startup time.

Example:

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8085
  periodSeconds: 5
  failureThreshold: 30
```

This allows up to 150 seconds for application startup before Kubernetes considers startup unsuccessful.

The probes have different responsibilities:

| Probe | Purpose |
|---|---|
| Startup | Allows sufficient time for application initialization |
| Liveness | Detects an application that needs restarting |
| Readiness | Determines whether a pod can receive traffic |

---

## 8. Kubernetes Services and Service Discovery

Each microservice has an internal ClusterIP Service.

| Service | Port | Type |
|---|---:|---|
| product-api-service | 8084 | ClusterIP |
| order-service | 8085 | ClusterIP |
| postgres-service | 5432 | ClusterIP |

The Order Service calls the Product API using Kubernetes service discovery:

```properties
PRODUCT_API_BASE_URL=http://product-api-service:8084
```

Because both applications are deployed in the same namespace, Kubernetes DNS resolves the Product API Service within that environment.

DEV applications communicate with DEV Services, and QA applications communicate with QA Services.

---

## 9. Ingress Configuration

The project uses an existing F5 NGINX Ingress Controller.

IngressClass:

```text
nginx
```

Controller identifier:

```text
nginx.org/ingress-controller
```

Controller namespace:

```text
nginx-ingress
```

The Ingress resources are maintained separately for DEV and QA.

### DEV routing

Host:

```text
dev.local
```

| Request path | Backend |
|---|---|
| /api/products | product-api-service:8084 |
| /api/orders | order-service:8085 |

### QA routing

Host:

```text
qa.local
```

| Request path | Backend |
|---|---|
| /api/products | product-api-service:8084 |
| /api/orders | order-service:8085 |

Example DEV Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: product-order-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: dev.local
      http:
        paths:
          - path: /api/products
            pathType: Prefix
            backend:
              service:
                name: product-api-service
                port:
                  number: 8084
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8085
```

The QA Ingress uses the same routing structure with `qa.local`.

---

## 10. Environment-Level Kustomization

Environment-level Kustomization combines the service overlays and Ingress resource.

DEV:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

resources:
  - ../../services/product-api/overlays/dev
  - ../../services/order-service/overlays/dev
  - ingress.yaml
```

QA:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: qa

resources:
  - ../../services/product-api/overlays/qa
  - ../../services/order-service/overlays/qa
  - ingress.yaml
```

This allows an entire environment to be deployed using a single Kustomize command.

---

## 11. Deployment Commands

Run commands from the project root directory.

### Preview DEV resources

```bash
kubectl kustomize environments/dev
```

### Preview QA resources

```bash
kubectl kustomize environments/qa
```

### Deploy DEV

```bash
kubectl apply -k environments/dev
```

### Deploy QA

```bash
kubectl apply -k environments/qa
```

### Check deployments

```bash
kubectl get deployments -n dev
kubectl get deployments -n qa
```

### Check pods

```bash
kubectl get pods -n dev
kubectl get pods -n qa
```

### Check Services

```bash
kubectl get services -n dev
kubectl get services -n qa
```

### Check Ingress

```bash
kubectl get ingress -A
```

### Check rollout status

```bash
kubectl rollout status deployment/product-api -n dev
kubectl rollout status deployment/order-service -n dev

kubectl rollout status deployment/product-api -n qa
kubectl rollout status deployment/order-service -n qa
```

---

## 12. Testing Ingress

The existing NGINX controller Service is ClusterIP.

For local testing, use port-forwarding.

Open a separate terminal:

```bash
kubectl port-forward -n nginx-ingress svc/nginx-ingress 8080:80
```

Keep this terminal running.

Open another terminal and test the Product API.

### DEV

```bash
curl -i -H "Host: dev.local" \
http://localhost:8080/api/products
```

### QA

```bash
curl -i -H "Host: qa.local" \
http://localhost:8080/api/products
```

### Order Service

```bash
curl -i -H "Host: dev.local" \
http://localhost:8080/api/orders
```

```bash
curl -i -H "Host: qa.local" \
http://localhost:8080/api/orders
```

The Order Service request path should match the actual controller mapping in the application.

A successful HTTP response confirms that the Ingress controller can route traffic to the corresponding Kubernetes Service and application.

---

## 13. Troubleshooting Commands

### Check pods

```bash
kubectl get pods -A
```

### Describe a pod

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### Application logs

```bash
kubectl logs deployment/product-api -n dev
kubectl logs deployment/order-service -n dev
```

### Previous container logs

```bash
kubectl logs <pod-name> -n dev --previous
```

### Check Service endpoints

```bash
kubectl get endpoints -n dev
kubectl get endpoints -n qa
```

### Check ConfigMaps

```bash
kubectl get configmaps -n dev
kubectl get configmaps -n qa
```

### Check Secrets

```bash
kubectl get secrets -n dev
kubectl get secrets -n qa
```

### Check Ingress controller logs

```bash
kubectl logs deployment/nginx-ingress -n nginx-ingress
```

### Check Ingress resources

```bash
kubectl describe ingress product-order-ingress -n dev
kubectl describe ingress product-order-ingress -n qa
```

---

## 14. Key Kubernetes Concepts Demonstrated

| Concept | Practical implementation |
|---|---|
| Namespace | DEV and QA isolation |
| Deployment | Spring Boot application deployment |
| ReplicaSet | Pod replica management |
| Pod | Application container execution |
| ClusterIP Service | Internal microservice communication |
| ConfigMap | Environment-specific configuration |
| Secret | Database credential management |
| StatefulSet | PostgreSQL workload |
| Persistent Volume Claim | PostgreSQL persistent storage |
| Startup Probe | Slow application startup handling |
| Liveness Probe | Application restart detection |
| Readiness Probe | Traffic readiness |
| Ingress | HTTP routing |
| Ingress Controller | NGINX reverse proxy implementation |
| Kustomize Base | Shared Kubernetes manifests |
| Kustomize Overlay | Environment-specific customization |

---

## 15. Running the Project

Prerequisites:

- Docker
- Kind
- kubectl
- Kustomize (or kubectl with built-in Kustomize support)
- jq
- An existing PostgreSQL deployment and database Secret
- An installed F5 NGINX Ingress Controller

Clone the repository:

```bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd product-order-kustomize
```

Deploy the DEV environment:

```bash
kubectl apply -k environments/dev
```

Deploy the QA environment:

```bash
kubectl apply -k environments/qa
```

Verify:

```bash
kubectl get pods -n dev
kubectl get pods -n qa
kubectl get ingress -A
```

**Note:** The PostgreSQL instance, its databases, namespace-specific Secrets, and Ingress Controller are prerequisites in this lab. They are not recreated by the environment-level Kustomizations.

---

## 16. Future Enhancements

- Package the application deployments as Helm charts.
- Introduce environment-specific Helm values.
- Automate deployments using CI/CD pipelines.
- Add TLS termination to Ingress.
- Implement NetworkPolicies for namespace traffic control.
- Introduce monitoring using Prometheus and Grafana.
- Integrate centralized logging.
- Introduce automated image version updates.

---

## Conclusion

This project demonstrates a practical Kubernetes deployment approach for multiple Spring Boot microservices across DEV and QA environments.

Kustomize provides reusable base manifests and environment-specific overlays, while Kubernetes Services enable internal communication and F5 NGINX Ingress Controller provides HTTP routing.

The project establishes a foundation for understanding production-oriented Kubernetes deployment practices before moving to Helm-based application packaging and release management.
