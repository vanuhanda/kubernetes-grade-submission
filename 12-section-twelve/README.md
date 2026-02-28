
# Repo Layout
```
11-HELM (API+Portal)
├── grade-submission-api/
│   │── templates/
│   │   ├── grade-submission-api-deployment.yaml
│   │   ├── grade-submission-api-secret.yaml
│   │   └── grade-submission-api-service.yaml
│   │── Chart.yaml
│   └── values.yaml 
│
├── grade-submission-portal/
│   │── templates/
│   │   ├── grade-submission-portal-config.yaml
│   │   ├── grade-submission-portal-deployment.yaml
│   │   ├── grade-submission-portal-secret.yaml
│   │   └── grade-submission-portal-service.yaml
│   │── Chart.yaml
│   │── grade-submission-portal-1.0.0.tgz
│   └── values.yaml
│
├── mongodb/
│   │── mongodb-secret.yaml
│   │── mongodb-service.yaml
│   └── mongodb-statefulset.yaml
│
├── Screenshots/
│   │── 1-helm-list-A.png
│   │── 2-pods-and-deployments.png
│   │── 3-uninstallation.png
│   │── 4-repackaging-reinstallation-verification.png
│   │── 5-helm-upgrade-grade-submission-api.png
│   │── 6-helm-upgrade-1.0.0.3-grade-submission-api.png
│   │── 7-rollback.png
│   │── 8-get-pods-error.png
│   │── 9-helm-upgrade-1.0.0.4-grade-submission-api.png
│   └── 10-section-twelve
│
└── README.md

```
# 🚀 Section 12 – Helm Upgrades, Namespaces & Rollbacks

## Overview

In this section, I moved Helm releases into the correct namespace, upgraded chart versions, performed rollbacks, and debugged a real production-style failure.

## 🔁 Helm Release Lifecycle Workflow

![alt text](Screenshots/10-section-twelve.png)


Helm Chart → Upgrade → Revision → Rollback → Namespace-scoped Resources


## 🏗️ Release Scope vs Resource Namespace

### 📌 Observation

When running:
```
helm list -A
```
![alt text](Screenshots/1-helm-list-A.png)

You’ll see releases initially deployed in the default namespace.

However, application resources were created inside:

```
kubectl get pods -n grade-submission
kubectl get deployments -n grade-submission
```

![alt text](Screenshots/2-pods-and-deployments.png)


### 🧠 Behind the Scenes

Helm release namespace and resource namespace are not automatically the same.

If you don’t specify -n, Helm installs the release in the default namespace, even if templates deploy resources elsewhere.

Best practice:

Keep release + resources in the same namespace.


## 🧹 Cleaning Up Incorrect Releases
```
helm uninstall grade-submission-api
helm uninstall grade-submission-portal
```
![alt text](Screenshots/3-uninstallation.png)


## 📦 Repackage and Install in Correct Namespace
```
helm package .
helm install grade-submission-api grade-submission-api-1.0.1.tgz -n grade-submission
```
Repeat for portal.
```
helm package .
helm install grade-submission-portal grade-submission-portal-1.0.0.tgz -n grade-submission
```

```
helm list -A
```
Everything correctly scoped to `grade-submission`.

![alt text](Screenshots/4-repackaging-reinstallation-verification.png)


# 🔄 Version Upgrades (Chart 1.0.2)

Updated:

- Image → stateless-v4
- Replaced env + configmap with 1MONGODB_URI`
- Used `stringData` in Secret
- Bumped Chart version → 1.0.2

Then: 

```
helm package .
helm upgrade grade-submission-api grade-submission-api-1.0.2.tgz -n grade-submission
```
![alt text](Screenshots/5-helm-upgrade-grade-submission-api.png)

# ⚡ Direct Upgrade Without Packaging (1.0.3)

Instead of packaging manually:

```
helm upgrade grade-submission-api . -n grade-submission
```
![12-section-twelve/Screenshots/6-helm-upgrade-1.0.3-grade-submission-api.png](Screenshots/6-helm-upgrade-1.0.3-grade-submission-api.png)

# 🧠 Behind the Scenes — Upgrade Flow

When running:

```
helm upgrade RELEASE .
```
Helm:

1. Reads Chart.yaml (new version)
2. Renders templates with values.yaml
3. Compares against existing release
4. Creates a new revision
5. Applies only necessary changes

Each upgrade increments a revision number.


# 🔙 Rollback Scenario

Assume version 1.0.3 fails.

```
helm rollback grade-submission-api 2 -n grade-submission
```
![alt text](Screenshots/7-rollback.png)

Helm restores previous revision state.


# ❌ Real Failure Scenario

After rollback, pods showed:

```
CreateContainerConfigError
```
![alt text](Screenshots/8-get-pods-error.png)

## 🧠 Root Cause

Deployment still referenced ConfigMap in envFrom, but ConfigMap had been removed earlier.

This caused container startup failure.

## 🛠 Fix Applied

Updated deployment:

Removed ConfigMap reference:

```
envFrom:
  - secretRef:
      name: "{{ .Values.microservice.name }}-secret"
```

Bumped chart version → 1.0.4

```
helm upgrade grade-submission-api . -n grade-submission
```
Pods running successfully again.

![alt text](Screenshots/9-helm-upgrade-1.0.0.4-grade-submission-api.png)

# Deployment Workflow

Helm deployment lifecycle:

1. Modify chart files

2. Preview using `helm template`

3. Upgrade using `helm upgrade`

4. Monitor via `kubectl`

5. Rollback if needed

6. Uninstall cleanly when required


# 📌 Key Takeaways

- Helm release namespace matters
- Chart versioning controls upgrade flow
- Every upgrade creates a new revision
- Rollback restores previous release state
- Removing referenced resources causes container config errors
- Always validate after upgrade


# 🏆 Why This Section Matters

This was the shift from:

Basic Helm usage → to → Real-world release management with upgrade & rollback strategy

This is how production-grade Kubernetes systems are maintained.



















We convert both:

- GRADE-SUBMISSION-API (Backend)
- GRADE-SUBMISSION-PORTAL (Frontend)

into production-ready Helm charts.

---

## 🏗️ Helm Workflow Architecture

![alt text](Screenshots/16-section-eleven.png)

## 📦 Why Helm?

Before Helm, we were manually applying:

- Deployments
- Services
- ConfigMaps
- Secrets
- Ingress

This approach works — but:

- It’s repetitive
- Hard to version
- Risky during cleanup
- Difficult to parameterize

Helm solves this by:
- Packaging everything into a Chart
- Centralizing configuration in `values.yaml`
- Enabling install / upgrade / uninstall with a single command
- Managing releases safely

# 🛠 Part 1 – Helm Installation Using Chocolatey (Windows)

```
choco install kubernetes-helm
```
![alt text](Screenshots/1-HELM-installation.png)

Verifying installation:

![alt text](Screenshots/2-HELM-version.png)

# 🧹 Cleanup Before Helm Migration
To prevent conflicts, remove previously deployed Kubernetes resources:
```
kubectl delete configmap,deployment,secret,service,ingress --all -n grade-submission
```
![alt text](Screenshots/3-clean-up.png)

MongoDB was accidentally deleted and re-applied:
```
MongoDB was accidentally deleted and re-applied:
```
![alt text](Screenshots/4-reapply-mongodb.png)


# 🧠 Helm Architecture Overview

Each Helm chart contains: 
```
    grade-submission-api/
    │── templates/
    │   ├── grade-submission-api-config.yaml
    │   ├── grade-submission-api-deployment.yaml
    │   ├── grade-submission-api-secret.yaml
    │   └── grade-submission-api-service.yaml
    │── Chart.yaml
    └── values.yaml 
```

Helm renders templates using:
```
helm template .
```


# 🔵 GRADE-SUBMISSION-API (Backend)

### 1️⃣ Chart.yaml

```
apiVersion: v2
name: grade-submission-api
description: Backend service for managing grade data
version: 1.0.0
```

### 2️⃣ values.yaml
We progressively evolved this file to include:

- Microservice configuration
- Container configuration
- Environment variables
- Secrets

Final structure:
```
microservice:
  name: grade-submission-api
  namespace: grade-submission
  replicas: 2

workload:
  image: rslim087/kubernetes-course-grade-submission-api:stateless-v3
  port: 3000
  resources:
    memory: "128Mi"
    cpu: "128m"
  livenessDelay: 15

env:
  MONGODB_HOST: mongodb
  MONGODB_PORT: "27017"

secrets:
  MONGODB_USER: YWRtaW4=
  MONGODB_PASSWORD: cGFzc3dvcmQxMjM=
```

### 3️⃣ Template Rendering
Render chart locally:
```
helm template .
```
This allows validation before installation.

### 4️⃣ Packaging the API Chart
```
helm package .
```
![alt text](Screenshots/5-helm-package-creation-api.png)

### 5️⃣ Installing the API Release
```
helm install grade-submission-api grade-submission-api-1.0.0.tgz
```
![alt text](Screenshots/6-helm-package-installation-api.png)

### 6️⃣ Verification
```
kubectl get pods -n grade-submission
```
![alt text](Screenshots/7-get-pods.png)
```
kubectl get configmap -n grade-submission
```
![alt text](Screenshots/8-get-configmap.png)
```
kubectl get secrets -n grade-submission
```
![alt text](Screenshots/9-get-secrets.png)

### 7️⃣ Uninstalling (Clean Removal if required)

```
helm uninstall grade-submission-api
```
![alt text](Screenshots/10-helm-package-uninstallation.png)

Helm removes:
- Deployment
- Service
- ConfigMap
- Secret
- Labels

No manual cleanup needed.

# 🟢 GRADE-SUBMISSION-PORTAL (Frontend)

The same Helm process was applied to the frontend.

### 1️⃣ values.yaml
```
microservice:
  name: grade-submission-portal
  namespace: grade-submission
  replicas: 1

workload:
  image: rslim087/kubernetes-course-grade-submission-portal
  port: 5001
  resources:
    memory: "128Mi"
    cpu: "128m"
  livenessDelay: 15

env:
  GRADE_SERVICE_HOST: grade-submission-api
```
### 2️⃣ Ingress Template
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "{{ .Values.microservice.name }}-ingress"
spec:
  ingressClassName: nginx
```

Ingress allows access via:
```
http://localhost
```
### 3️⃣ Template Validation
```
helm template .
```

### 4️⃣ Packaging Portal Chart
```
helm package .
```
![alt text](Screenshots/11-helm-package-creation-portal.png)


### 5️⃣ Installing Portal Release
```
helm install grade-submission-portal grade-submission-portal-1.0.0.tgz
```
![alt text](Screenshots/12-helm-package-installation-portal.png)


### 6️⃣ Deployment Verification
```
kubectl get deployments -n grade-submission
```

![alt text](Screenshots/13-final-deployments.png)

# 🌐 UI Validation

After installation:

Visit:
```
http://localhost
```
Portal loads successfully.

![alt text](Screenshots/14-form.png)

![alt text](Screenshots/15-grades.png)


# 🔍 Behind the Scenes – What Helm Is Really Doing

When you run: 
```
helm install grade-submission-api ...
```
Helm:

- Renders templates using values.yaml
- Creates Kubernetes manifests
- Labels them with release metadata
- Stores release state in Kubernetes
- Enables version-controlled upgrades


When you run:
```
helm uninstall grade-submission-api
```

Helm:

- Tracks all created resources
- Deletes them safely
- Prevents accidental removal of unrelated objects

# 📊 What We Achieved in Section 11
```
| Feature          | Before Helm | After Helm       |
| ---------------- | ----------- | ---------------- |
| Deployment       | Manual      | Packaged         |
| Cleanup          | Risky       | One command      |
| Versioning       | None        | Chart versioning |
| Parameterization | Hardcoded   | values.yaml      |
| Reusability      | Low         | High             |
| Release Tracking | None        | Helm Release     |
```
# 🏁 Final Architecture After Helm
```
Ingress
   ↓
Portal (Helm Release)
   ↓
API (Helm Release)
   ↓
MongoDB (StatefulSet)
```
Two independent Helm releases:

- grade-submission-api
- grade-submission-portal

# 🧠 Key Concepts Learned in Section 11

Helm acts as a package manager for Kubernetes, allowing multiple Kubernetes resources to be bundled and managed as a single release.

Instead of managing individual YAML files (Deployment, Service, ConfigMap, Secret, Ingress), Helm groups them into a Chart, enabling:

- Centralized configuration
- Versioned deployments
- Simplified upgrades and rollbacks
- Clean uninstall of all related resources
- Release tracking

This significantly improves operational control compared to applying loose manifests.

# 📂 Helm Chart Structure
Each service was converted into the following structure:

```
📂 Helm Chart Structure

  grade-submission-api/
  ├── Chart.yaml
  ├── values.yaml
  └── templates/
```

Chart.yaml

Contains metadata such as chart name and version.

values.yaml

Holds default configuration values (replicas, image, ports, environment variables, secrets).

templates/

Contains Kubernetes manifest templates using Helm templating syntax.

### ⚙️ Common Helm Commands Used

Install a release:
```
helm install grade-submission-api grade-submission-api-1.0.0.tgz
```
Uninstall a release:
```
helm uninstall grade-submission-api
```
Upgrade a release:
```
helm upgrade grade-submission-api .
```
Render templates locally:
```
helm template .
```
Package a chart: 
```
helm package .
```

# 🎯 Section 11 Summary

This section transforms the project from:

“YAML-based Kubernetes deployment”

into:

“Versioned, parameterized, production-style Helm-managed releases”

Helm now manages:

- Installations
- Rollbacks
- Upgrades
- Clean uninstalls
- Release history

This is how real-world Kubernetes environments operate.

