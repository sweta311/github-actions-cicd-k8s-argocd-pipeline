# WizSuite Web API

![CI/CD Pipeline](https://github.com/sweta311/github-actions-cicd-k8s-argocd-pipeline/actions/workflows/main.yaml/badge.svg)

This repository contains the source code for the **WizSuite Web API**, a Node.js application.

It is integrated with a fully automated CI/CD pipeline using **GitHub Actions** that follows GitOps principles. Pushing code to the `prod` branch automatically builds a new Docker image, pushes it to Docker Hub, and updates the deployment configuration in a separate Kubernetes manifests repository. **ArgoCD** then detects this change and deploys the new version to the production Kubernetes cluster.

## Architecture & Workflow

The deployment process is fully automated and designed to be seamless from code commit to production deployment.

```
+-------------------+      +----------------------+      +-----------------------------+      +-------------------+
|                   |      |                      |      |                             |      |                   |
|  Developer        +----->+  GitHub (This Repo)  +----->+     GitHub Actions CI/CD    +----->+   Docker Hub      |
|  (git push)       |      |  (prod branch)       |      |  (Build, Push, Update)      |      |  (Image Registry) |
|                   |      |                      |      |                             |      |                   |
+-------------------+      +----------------------+      +--------------+--------------+      +---------+---------+
                                                                         |                                |
                                                                         | (Updates Manifest File)        | (Pulls Image)
                                                                         v                                v
+-------------------+      +----------------------+      +-----------------------------+      +-------------------+
|                   |      |                      |      |                             |      |                   |
|  ArgoCD           +<-----+  GitHub (Manifests)   +<-----+                             |      |  Kubernetes Cluster|
|  (Syncs State)    |      |  (Detects changes)   |      |                             |      |  (Pulls & Runs)   |
|                   |      |                      |      |                             |      |                   |
+-------------------+      +----------------------+      +-----------------------------+      +-------------------+
```

The end-to-end workflow is as follows:
1.  A developer pushes new code or changes to the `prod` branch of this repository.
2.  The **GitHub Actions workflow** is automatically triggered.
3.  **Build**: The action builds a new Docker image from the `Dockerfile`. The image is tagged with the unique Git commit SHA for traceability (e.g., `your-user/your-repo:a1b2c3d4`).
4.  **Publish**: The new Docker image is pushed to a Docker Hub repository.
5.  **Update Manifests**: The action checks out a separate Git repository that holds the Kubernetes manifests (`Wizsuite-2.0-K8s-manifest.git`).
6.  It then uses `sed` to update the `image` tag in the `wizsuite-deploy.yaml` file with the new Docker image tag.
7.  **Commit & Push**: The action commits and pushes the updated manifest file back to the manifest repository.
8.  **ArgoCD**, which is continuously monitoring the manifest repository, detects the change.
9.  **Sync**: ArgoCD automatically syncs the state, pulling the new image and applying the updated configuration to the Kubernetes cluster, resulting in a rolling update of the application.

## Core Components

### 1. Application (`Dockerfile`)

This is a standard Node.js application. The `Dockerfile` creates an optimized image by:
*   Using an official Node.js base image.
*   Copying only `package.json` first to leverage Docker's build cache for dependencies.
*   Copying the application source code.
*   Setting the entrypoint to run the application via `node /app/app.js`.

### 2. CI/CD Pipeline (`.github/workflows/publish.yml`)

This workflow automates the entire build and deploy process. It is triggered on a `push` to the `prod` branch or can be run manually (`workflow_dispatch`).

**Key Steps:**
*   **Checkout**: Checks out the application source code.
*   **Build & Publish**: Builds the Docker image and pushes it to Docker Hub using credentials stored in GitHub Secrets.
*   **Clone & Update Manifests**: Clones the Kubernetes manifest repository, updates the image tag in the deployment YAML, and pushes the changes back. This is the core of the GitOps "push-style" update.

### 3. Kubernetes Manifests

The desired state of our application in Kubernetes is defined in a separate repository: `https://github.com/xyz/Wizsuite-2.0-K8s-manifest.git`. This includes:
*   `wizsuite-namespace.yaml`: Defines the `wizsuite-prod` namespace to isolate our application.
*   `wizsuite-deploy.yaml`: The Deployment object that manages the application pods. **This is the file our CI/CD pipeline updates.**
*   `wizsuite-clusterip.yaml`: A Service to provide a stable internal endpoint for the application.
*   `wizsuite-configmap.yaml`: Stores non-sensitive configuration data for the application.
*   `wizsuite-ingress.yaml`: Manages external access to the application, routing traffic from `ind-prod-web.xyz.com` to the service.
*   `wizsuite-certificate.yaml`: An instruction for `cert-manager` to automatically provision a TLS certificate for our domain.

### 4. ArgoCD Application

ArgoCD is the GitOps continuous delivery tool that completes the cycle. It is configured to:
*   **Watch** the `https://github.com/xyz/Wizsuite-2.0-K8s-manifest.git` repository.
*   **Target** the `wizsuite-prod` namespace in the Kubernetes cluster.
*   **Automatically Sync** any changes found in the Git repository to the live cluster, ensuring the cluster's state always matches the desired state defined in Git.

## Setup & Prerequisites

To use this CI/CD pipeline, the following **GitHub Secrets and Variables** must be configured in this repository's settings (`Settings > Secrets and variables > Actions`):

### Secrets
*   `DOCKER_USERNAME`: Your Docker Hub username.
*   `DOCKER_PASSWORD`: Your Docker Hub password or access token.
*   `GH_TOKEN`: A GitHub Personal Access Token (PAT) with `repo` scope or, for better security, `contents: write` permission specifically on the manifest repository. This is used to push the updated manifest file.

### Variables
*   `MANIFEST_REPOSITORY`: The full name of your Kubernetes manifest repository (e.g., `your-org/Wizsuite-2.0-K8s-manifest`).
*   `MANIFEST_FILE_NAME_WEBAPI`: The path to the deployment file within the manifest repository that needs to be updated (e.g., `wizsuite-deploy.yaml`).

## How to Deploy

1.  Commit and push your changes to the `prod` branch.
    ```bash
    git push origin prod
    ```
2.  The GitHub Actions workflow will automatically start. You can monitor its progress in the "Actions" tab of this repository.
3.  Once the workflow succeeds, ArgoCD will detect the change and begin deploying the new version to the cluster. This can be monitored from the ArgoCD UI.

---

### Argocd Sample 

<img width="1365" height="586" alt="Screenshot from 2025-07-29 11-27-55" src="https://github.com/user-attachments/assets/4fc475e8-fc85-4a3b-a81c-57b0893d7a35" />

### Workflows Job Sample

<img width="1360" height="587" alt="Screenshot from 2025-07-29 11-26-52" src="https://github.com/user-attachments/assets/bb81e0d7-b647-4b32-8a94-af4fef0a7f6e" />

