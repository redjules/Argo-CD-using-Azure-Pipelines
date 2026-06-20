# 🚀 Azure DevOps CI/CD with GitOps (Argo CD)

> Part 1 covered Continuous Integration (ci-voting-app repo)  building Docker images and pushing to Azure Container Registry (ACR). This guide covers **Continuous Delivery** using a GitOps approach with **Argo CD** on **Azure Kubernetes Service (AKS)**.

---

## 📋 Table of Contents

- [🗺️ Project Overview](#️-project-overview)
- [🏗️ Application Architecture](#️-application-architecture)
- [✅ Prerequisites](#-prerequisites)
- [☁️ Step 1 — Create an AKS Cluster](#️-step-1--create-an-aks-cluster)
- [🔌 Step 2 — Connect to the Cluster](#-step-2--connect-to-the-cluster)
- [⚙️ Step 3 — Install Argo CD](#️-step-3--install-argo-cd)
- [🖥️ Step 4 — Access the Argo CD UI](#️-step-4--access-the-argo-cd-ui)
- [🔗 Step 5 — Connect Argo CD to Azure Repos](#-step-5--connect-argo-cd-to-azure-repos)
- [📦 Step 6 — Create an Argo CD Application](#-step-6--create-an-argo-cd-application)
- [📝 Step 7 — Write the Update Script](#-step-7--write-the-update-script)
- [🔧 Step 8 — Add the Update Stage to Azure Pipelines](#-step-8--add-the-update-stage-to-azure-pipelines)
- [🔐 Step 9 — Configure Image Pull Secret](#-step-9--configure-image-pull-secret)
- [⏱️ Step 10 — Tune Argo CD Reconciliation Interval](#️-step-10--tune-argo-cd-reconciliation-interval)
- [🤔 Why GitOps?](#-why-gitops)
- [🛠️ Troubleshooting](#️-troubleshooting)
- [🔗 Resources](#-resources)

---

## 🗺️ Project Overview

This project demonstrates an **end-to-end CI/CD pipeline** for a multi-microservice voting application originally developed by the Docker samples team. The app lets users vote between two options and displays live results.

### 🧩 Microservices

| Service     | Language       | Role                                 |
| ----------- | -------------- | ------------------------------------ |
| `vote` 🗳️   | Python (Flask) | Voting UI                            |
| `worker` ⚙️ | .NET           | Reads from Redis, writes to Postgres |
| `result` 📊 | Node.js        | Displays results                     |
| `redis` ⚡  | —              | In-memory data store                 |
| `db` 🗄️     | PostgreSQL     | Persistent vote storage              |

---

## 🏗️ Application Architecture

```
👨‍💻 Developer
      │
      ▼ (git push)
📁 Azure Repos ────────────────────────────────────────────┐
      │                                                     │
      ▼ (trigger)                                          │ (watches for changes)
🔄 Azure Pipelines                                         │
      ├── 🔨 Stage 1: Build   → Docker image created       │
      ├── 📤 Stage 2: Push    → Image pushed to ACR        │
      └── ✏️  Stage 3: Update  → K8s manifest updated ──────┘
                                                           │
                                                    🐙 Argo CD (GitOps)
                                                           │
                                                           ▼
                                                  ☸️  AKS Cluster
                                               (pods auto-updated ✅)
```

> 💡 Argo CD is the **only** component that touches the Kubernetes cluster. There is **no direct connection** between the Azure Pipeline and the cluster, the **Git repository acts as the single source of truth**.

---

## Prerequisites

Before you begin, make sure you have the following ready:

- **Azure CLI** installed and configured
- **`kubectl`** installed
- **Azure DevOps organisation** with source code in Azure Repos
- **Azure Container Registry (ACR)** set up _(covered in Part 1)_
- **Self-hosted Azure Pipelines agent** _(required for free-tier subscriptions)_

---

## ☁️ Step 1 — Create an AKS Cluster

1. In the Azure Portal, search for **Kubernetes services** and click **Create**.
2. Select your resource group (e.g. `azure-cicd`).
3. Use the following recommended settings for demos/dev-test:

   | Setting                   | Value                                         |
   | ------------------------- | --------------------------------------------- |
   | Cluster preset            | Dev/Test                                      |
   | Region                    | West US 2 _(or nearest with available quota)_ |
   | Availability zones        | Any single zone                               |
   | Scale method              | Autoscale                                     |
   | Min nodes                 | `1`                                           |
   | Max nodes                 | `2`                                           |
   | Enable public IP per node | ✅                                            |

4. Click **Review + Create** → **Create**. ⏳ The cluster takes ~10–15 minutes to provision.

> ⚠️ **Tip:** If you hit resource quota errors, try switching to a different Azure region (e.g. West US 2 instead of East US).

---

## 🔌 Step 2 — Connect to the Cluster

```bash
az aks get-credentials \
  --name <your-cluster-name> \
  --resource-group azure-cicd \
  --overwrite-existing

#  Verify connection
kubectl get pods
```

---

## ⚙️ Step 3 — Install Argo CD

```bash
#  Create the Argo CD namespace
kubectl create namespace argocd

#  Install Argo CD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

#  Wait for pods to be ready
kubectl get pods -n argocd
```

---

## 🖥️ Step 4 — Access the Argo CD UI

### 🔑 Get the Admin Password

```bash
# Retrieve and decode the initial admin secret
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### 🌐 Expose the Argo CD Server via NodePort

```bash
kubectl edit svc argocd-server -n argocd
# ✏️ Change `type: ClusterIP` to `type: NodePort`

# Get the assigned NodePort
kubectl get svc -n argocd

# Get the node's external IP
kubectl get nodes -o wide
```

### 🔓 Open the Firewall Port

1. In the Azure Portal, go to **Virtual Machine Scale Sets (VMSS)**.
2. Select your node pool → **Instances** → **Networking**.
3. Add an **inbound port rule** for the NodePort assigned to Argo CD (e.g. `30461`).

🌍 Access the UI at: `http://<NODE_EXTERNAL_IP>:<NODEPORT>`

| Field    | Value                           |
| -------- | ------------------------------- |
| Username | `admin`                         |
| Password | `<decoded password from above>` |

---

## 🔗 Step 5 — Connect Argo CD to Azure Repos

1.  Generate a **Personal Access Token (PAT)** in Azure DevOps:

- Go to **User Settings → Personal Access Tokens → New Token**
- Grant at minimum **Read** access to Code _(Argo CD only needs to pull)_

2. In the Argo CD UI: **Settings → Repositories → Connect Repo (HTTPS)**

3. Build the repository URL by replacing your organisation name with the PAT:

   ```
   https://<YOUR_PAT>@dev.azure.com/<ORG>/<PROJECT>/_git/<REPO>
   ```

4. Click **Connect** — status should show ✅ **Successful**

---

## 📦 Step 6 — Create an Argo CD Application

In the Argo CD UI, click **New App** and fill in:

| Field            | Value                                  |
| ---------------- | -------------------------------------- |
| Application name | `vote-app`                             |
| Project          | `default`                              |
| Sync policy      | ⚡ **Automatic**                       |
| Repository URL   | _(auto-populated from connected repo)_ |
| Path             | `k8s-specifications`                   |
| Cluster          | _(default — same cluster)_             |
| Namespace        | `default`                              |

Click **Create**. Argo CD will immediately sync and deploy all manifests found in the specified path.

---

## 📝 Step 7 — Write the Update Script

Create `scripts/update-k8s-manifest.sh` in your Azure Repo:

```bash
#!/bin/bash
set -x

# 📌 Arguments:
#   $1 - Service name prefix  (e.g. "vote", "result")
#   $2 - ACR repository name  (e.g. "votingapp", "resultapp")
#   $3 - New image tag        (e.g. the pipeline build number)

REPO_URL="https://<YOUR_PAT>@dev.azure.com/<ORG>/<PROJECT>/_git/<REPO>"

git clone "$REPO_URL" /tmp/repo
cd /tmp/repo

# ✏️ Replace the image tag in the deployment manifest
sed -i "s|image:.*|image: <YOUR_ACR_NAME>.azurecr.io/$2:$3|g" \
  k8s-specifications/$1-deployment.yaml

git config user.email "pipeline@ci"
git config user.name  "Azure Pipeline"
git add .
git commit -m "🔖 Update $1 image to tag $3 [skip ci]"
git push
```

> 🔐 **Security note:** In production, pass the PAT as a **pipeline secret variable** rather than hard-coding it in the script.

---

## 🔧 Step 8 — Add the Update Stage to Azure Pipelines

Add a third stage to your `azure-pipelines.yml` after the push stage:

```yaml
- stage: Update
  displayName: ✏️ Update K8s Manifest
  jobs:
    - job: UpdateManifest
      steps:
        - task: ShellScript@2
          inputs:
            scriptPath: scripts/update-k8s-manifest.sh
            args: "vote $(imageRepository) $(Build.BuildId)"
```

### 📌 Variable Reference

| Variable             | Description                                           |
| -------------------- | ----------------------------------------------------- |
| `$(imageRepository)` | ACR repository name _(defined in pipeline variables)_ |
| `$(Build.BuildId)`   | Auto-incremented build number used as the image tag   |

### 🔄 What Happens on Each Pipeline Run

```
1.  Build    → Docker image created
2.  Push     → Image pushed to ACR with build ID tag
3.  Update   → deployment.yaml updated in Git with new tag
4.  Argo CD  → Detects Git change, redeploys to AKS automatically
```

---

## 🔐 Step 9 — Configure Image Pull Secret

Kubernetes needs credentials to pull images from your **private ACR**.

### 👤 Enable ACR Admin User

In the Azure Portal → **Container Registry → Access keys** → enable **Admin user**. Note the **username** and **password**.

### 🔑 Create the Kubernetes Secret

```bash
kubectl create secret docker-registry acr-secret \
  --namespace default \
  --docker-server=<YOUR_ACR_NAME>.azurecr.io \
  --docker-username=<ACR_ADMIN_USERNAME> \
  --docker-password=<ACR_ADMIN_PASSWORD>
```

### 📄 Reference the Secret in Your Deployment Manifest

In `k8s-specifications/vote-deployment.yaml`, add `imagePullSecrets` at the pod spec level:

```yaml
spec:
  containers:
    - name: vote
      image: <YOUR_ACR_NAME>.azurecr.io/votingapp:latest
  imagePullSecrets:
    - name: acr-secret # 🔑 matches the secret name above
```

Commit this change — Argo CD will automatically apply it to the cluster ✅

> ⚠️ **Common mistake:** Make sure you create the secret in the **`default` namespace** (where your pods run), not in the `argocd` namespace!

---

## ⏱️ Step 10 — Tune Argo CD Reconciliation Interval

By default, Argo CD polls for Git changes every **3 minutes**. Reduce this for faster deployments:

```bash
kubectl edit cm argocd-cm -n argocd
```

Add the following under `data`:

```yaml
data:
  timeout.reconciliation: 10s # ⚡ was 180s by default
```

Save and exit. Argo CD will now detect Git changes within **~10 seconds** ⚡

> ⚠️ **Caution:** Don't set this too low (e.g. 2–3 s) in environments with many microservices — it increases load on both Argo CD and your Git server.

---

## 🤔 Why GitOps?

| Traditional CD (scripts/Ansible)    | GitOps (Argo CD)                                 |
| ----------------------------------- | ------------------------------------------------ |
| Pipeline pushes directly to cluster | Git is the **single source of truth**            |
| Drift is silent and undetected      | Drift is **detected and auto-corrected** 🔄      |
| Manual rollback = re-run a pipeline | Rollback = **revert a Git commit** 🕐            |
| Cluster state is hard to audit      | Full **audit trail in Git history** 📋           |
| Config scattered across pipelines   | Everything **declarative in version control** 📁 |

### 🛡️ Drift Correction in Action

> Someone manually changes the running image tag directly on the Kubernetes cluster...
>
> 🐙 Argo CD detects the mismatch with Git → **automatically reverts the change** within the reconciliation window. No human intervention needed!

---

## 🛠️ Troubleshooting

| ⚠️ Symptom                              | 🔍 Likely Cause                               | ✅ Fix                                                   |
| --------------------------------------- | --------------------------------------------- | -------------------------------------------------------- |
| `ImagePullBackOff`                      | Secret created in wrong namespace             | Re-create `acr-secret` in the **`default`** namespace    |
| Image name malformed in deployment YAML | Missing `.azurecr.io` suffix in update script | Update the full registry URL in `update-k8s-manifest.sh` |
| Argo CD not syncing                     | Reconciliation interval too long              | Set `timeout.reconciliation: 10s` in `argocd-cm`         |
| Pipeline variables not expanding        | Wrong variable syntax in `args`               | Use `$(varName)` syntax in the YAML `args` field         |
| AKS creation fails with quota error     | Insufficient regional quota                   | Switch to a different Azure region (e.g. West US 2)      |
| App not accessible in browser           | NodePort not open in NSG                      | Add inbound rule for the NodePort in VMSS → Networking   |

---

## 🔁 Extending to Other Microservices

The update script is **parameterised**, adding a new service is straightforward. Just add another step in your pipeline with different arguments:

```yaml
#  Vote service
args: 'vote $(imageRepository) $(Build.BuildId)'

#  Result service
args: 'result $(resultImageRepository) $(Build.BuildId)'
```

No changes to the shell script needed! 🎉

---

## 🔗 Resources

- 🐙 [Argo CD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting-started/)
- ☸️ [AKS Documentation](https://learn.microsoft.com/en-us/azure/aks/)
- 📦 [Azure Container Registry Docs](https://learn.microsoft.com/en-us/azure/container-registry/)
- 🔄 [Azure Pipelines YAML Reference](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/)
- 🗂️ Architecture diagram: available in the `day15/` folder of the course repository

---

> 💬 _If you've implemented the Result service as well, drop a comment and let the community know!_  
> ⭐ _Found this helpful? Star the repo and share it with your network!_
