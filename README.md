# GitHub Actions POC

GitHub Actions is a built-in CI/CD platform available in Gituhub, used to automate build and deployment.

All workflow files are in [.github/workflows/](.github/workflows/)

## Workflow details

### 1. GitHub-hosted runner

Builds on a GitHub-hosted `ubuntu-latest` runner and always deploys the package to GitHub Packages, using [settings.xml](settings.xml) for credentials.

![GitHub-hosted runner workflow](images/workflow-github-hosted.svg)

### 2. Self-hosted runner (Linux)

The same build on a self-hosted runner with the `Linux` label. It deploys to GitHub Packages only when `UploadToPackage` is ticked.

![Self-hosted Linux workflow](images/workflow-selfhosted-linux.svg)

### 3. Self-hosted runner (Windows)

The same build on a self-hosted runner with the `Windows` label. Tests are skipped (`-DskipTests`) in both the build and the deploy step.

![Self-hosted Windows workflow](images/workflow-selfhosted-windows.svg)

### 4. Build and upload to a Nexus repository

Deploys with [settingsNexus.xml](settingsNexus.xml), which points at a Nexus 3 instance on `localhost:8081` (`maven-releases` / `maven-snapshots`). [pomNexus.xml](pomNexus.xml) contains the matching `distributionManagement` block.

![Nexus workflow](images/workflow-nexus.svg)

### 5. Docker build and push to GitHub Container Registry

A two-job pipeline. The `build` job packages the JAR and uploads it as a build artifact. The `docker` job (`needs: build`) downloads the JAR, builds the image with the [Dockerfile](Dockerfile), scans it with Trivy, and pushes `ghcr.io/<owner>/myjavapp:<run_number>`.

![Docker to GHCR workflow](images/workflow-docker-ghcr.svg)

### 6. Docker build and push to Azure Container Registry

The same two-job layout, except the `docker` job signs in to Azure and pushes to ACR. An Azure Container Instances deploy step is in the file but commented out.

![Docker to ACR workflow](images/workflow-docker-acr.svg)

### 7. Upstream and downstream workflows

The **Upstream** workflow builds the JAR, optionally deploys it to GitHub Packages, and uploads it as an artifact. When Upstream completes, GitHub starts the **Downstream** workflow (`on: workflow_run`). Downstream uses `github.event.workflow_run.id` to download the artifact from that upstream run, then builds, scans and pushes the Docker image to GHCR.

![Upstream and downstream workflows](images/workflow-upstream-downstream.svg)

### 8. Self-hosted runners on Kubernetes (ARC)

Runs on `arc-runner-set`, a runner scale set that [Actions Runner Controller](https://github.com/actions/actions-runner-controller) manages in an AKS cluster. Each job gets a new, ephemeral runner pod. This POC workflow only checks out the repository, to prove that jobs reach the cluster. See [Kubernetes runners](SELF_HOSTED_RUNNERS.MD#kubernetes-runners-actions-runner-controller) for the cluster setup and how to use a custom runner image.

## Workflows Overview

| Workflow | File | Trigger | Runner | Output |
|---|---|---|---|---|
| My Java App Workflow | [buildWorkflow.yml](.github/workflows/buildWorkflow.yml) | manual | `ubuntu-latest` | Maven package → GitHub Packages |
| My Java App Workflow (Selfhosted) | [buildWorkflow _selfHosted_Linux.yml](.github/workflows/buildWorkflow%20_selfHosted_Linux.yml) | manual, `UploadToPackage` | self-hosted `Linux` | Maven package → GitHub Packages *(optional)* |
| My Java App Workflow (Selfhosted-Windows) | [buildWorkflow _selfHostedWindows.yml](.github/workflows/buildWorkflow%20_selfHostedWindows.yml) | manual, `UploadToPackage` | self-hosted `Windows` | Maven package → GitHub Packages *(optional)* |
| My Java App Workflow (Selfhosted-Nexus) | [buildWorkflow _selfHosted_Nexus.yml](.github/workflows/buildWorkflow%20_selfHosted_Nexus.yml) | manual, `UploadToPackage` | self-hosted `Linux` | Maven package → Nexus *(optional)* |
| My Java App Workflow (Docker) | [buildWorkflow _selfHosted_Docker.yml](.github/workflows/buildWorkflow%20_selfHosted_Docker.yml) | manual, `UploadToRegistry` | self-hosted `Linux` | Docker image → GHCR *(optional)* |
| My Java App Workflow (ACR) | [buildWorkflow _selfHosted_ACR.yml](.github/workflows/buildWorkflow%20_selfHosted_ACR.yml) | manual, `UploadToRegistry` | self-hosted `Linux` | Docker image → ACR *(optional)* |
| My Java App Workflow (Upstream) | [buildWorkflow _upStream.yml](.github/workflows/buildWorkflow%20_upStream.yml) | manual, `UploadToPackage` | `ubuntu-latest` | JAR artifact, GitHub Packages *(optional)* |
| My Java App Workflow (Downstream) | [buildWorkflow _downStream.yml](.github/workflows/buildWorkflow%20_downStream.yml) | `workflow_run` of Upstream | `ubuntu-latest` | Docker image → GHCR |
| My App (k8s Selfhosted) | [buildWorkflow _selfHosted_k8s.yml](.github/workflows/buildWorkflow%20_selfHosted_k8s.yml) | manual | ARC scale set `arc-runner-set` (AKS) | Checkout only *(POC)* |

*Optional* means that step or job runs only when you tick the boolean input when you start the workflow. In the diagrams below, dashed boxes and arrows mark these conditional parts.

## Self-hosted runner setup

To set up the Linux, Windows and Kubernetes (ARC) runners these workflows need, see [SELF_HOSTED_RUNNERS.MD](SELF_HOSTED_RUNNERS.MD).

## Secrets and configuration

| Secret | Used by | Purpose |
|---|---|---|
| `GITHUB_TOKEN` *(provided automatically)* | GitHub Packages and GHCR workflows | Maven deploy and `docker login ghcr.io` |
| `NEXUS_USERNAME`, `NEXUS_PASSWORD` | Selfhosted-Nexus | Nexus credentials referenced by `settingsNexus.xml` |
| `AZURE_CREDENTIALS` | ACR | Service principal JSON for `azure/login` |
| `REGISTRY_LOGIN_SERVER`, `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` | ACR | ACR login server and credentials |

The Kubernetes runners don't use repository secrets. ARC authenticates with a GitHub App whose credentials are stored as the Kubernetes secret `arc-github-app` (see [step 4](SELF_HOSTED_RUNNERS.MD#4-store-the-github-app-credentials-as-a-secret)).

To push to GitHub Packages or GHCR, the repository's **Settings → Actions → General → Workflow permissions** must allow *Read and write*.
