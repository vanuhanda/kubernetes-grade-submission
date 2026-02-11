
# Repo Layout
```
08-section-eight-Config Maps and Secrets

├── grade-submission-api/
│   ├── grade-submission-api-config.yaml
│   ├── grade-submission-api-secret.yaml
│   ├── grade-submission-api-deployment.yaml
│   └── grade-submission-api-service.yaml
│
├── grade-submission-portal/
│   ├── grade-submission-portal-config.yaml
│   ├── grade-submission-portal-deployment.yaml
│   └── grade-submission-portal-service.yaml
│
├── mongodb/
│   ├── mongodb-secret.yaml
│   ├── mongodb-statefulset.yaml
│   └── mongodb-service.yaml
│
├── Screenshots
│   ├── base-64.png
│   ├── grade-api.png
│   ├── grade-portal.png
│   ├── kubectl-apply-api.png
│   ├── kubectl-apply-mongodb.png
│   ├── kubectl-apply-portal.png
│   └── section-eight.png
│
└── README.md
```


# Section 07 - ConfigMaps & Secrets (Decoupling Configuration from Code)

## Overview
This section focuses on externalizing configuration and secrets from application manifests using ConfigMaps and Secrets, following Kubernetes best practices for security, portability, and maintainability.

By the end of this section:

- Application code remains unchanged

- Configuration can be updated independently

- Sensitive values are not hardcoded in manifests

- Workloads are environment-agnostic

---

## Application Architecture


```
External User
     |
     |  NodePort :32001
     v
Grade Submission Portal (Deployment)
     |
     |  Service (ClusterIP)
     v
Grade Submission API (Deployment)
     |
     |  Service (ClusterIP)
     v
MongoDB (StatefulSet + PVC)

```
### Configuration Flow

## Frontend

- Reads backend hostname from ConfigMap

## Backend

- Reads DB host/port from ConfigMap

- Reads DB credentials from Secret

## Database

- Reads root credentials from Secret

- Persists data using PVC

![alt text](Screenshots/section-eight.png)


### What Changed in This Section

## 1️⃣ Environment Variables Removed from Deployments

- No more hardcoded `env:` blocks

- All configuration moved to:

  - ConfigMaps → non-sensitive data

  - Secrets → credentials


## 2️⃣ ConfigMaps Introduced

Backend ConfigMap
```
kind: ConfigMap
data:
  MONGODB_HOST: mongodb
  MONGODB_PORT: "27017"
```

Frontend ConfigMap
```
kind: ConfigMap
data:
  GRADE_SERVICE_HOST: grade-submission-api
```

## 3️⃣ Secrets Introduced (Base64 Encoded)
```
echo -n 'admin' | base64
echo -n 'password123' | base64
```
```
kind: Secret
type: Opaque
data:
  MONGODB_USER: YWRtaW4=
  MONGODB_PASSWORD: cGFzc3dvcmQxMjM=
```
![alt text](Screenshots/base-64.png)

## 4️⃣ envFrom Used in Pods
```
envFrom:
  - configMapRef:
      name: grade-submission-api-config
  - secretRef:
      name: grade-submission-api-secret
```
This keeps:

- YAML clean

- Configuration reusable

- Secrets invisible in pod specs

## ▶️ Deployment Order (Important)

```
kubectl apply -f mongodb/
kubectl apply -f grade-submission-api/
kubectl apply -f grade-submission-portal/
```


## 🔍 Verification

```
kubectl get pods -n grade-submission
kubectl logs -f <api-pod> -n grade-submission
```

Expected Logs:
```
Attempting to connect to MongoDB...
Connected to MongoDB
```
![alt text](Screenshots/kubectl-apply-mongodb.png)
![alt text](Screenshots/kubectl-apply-api.png)
![alt text](Screenshots/kubectl-apply-portal.png)

# 🧠 Behind the Scenes (What Kubernetes Does)

ConfigMaps

- Inject values as environment variables at runtime

- Allow config changes without rebuilding images

- Enable environment-specific overrides


Secrets

- Stored separately from application code

- Base64 encoded (⚠️ not encrypted)

- Mounted securely into pods

Why This Matters

- Clean separation of code vs configuration

- Safer credential handling

- Easier CI/CD and environment promotion

- Production-aligned Kubernetes design

⚠️ Security Note

Kubernetes Secrets are not encrypted by default.

In real environments:

- Enable encryption at rest

- Use external secret managers (Vault, AWS Secrets Manager, etc.)

- Restrict RBAC access to Secrets

✅ Key Takeaways

- ConfigMaps handle configuration

- Secrets handle credentials

- Deployments should never contain secrets directly

- Kubernetes encourages declarative, portable, secure design

