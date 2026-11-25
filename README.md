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

*Optional* means that step or job runs only when you tick the boolean input when you start the workflow. In the diagrams below, dashed boxes and arrows mark these conditional parts.

## Secrets and configuration

| Secret | Used by | Purpose |
|---|---|---|
| `GITHUB_TOKEN` *(provided automatically)* | GitHub Packages and GHCR workflows | Maven deploy and `docker login ghcr.io` |
| `NEXUS_USERNAME`, `NEXUS_PASSWORD` | Selfhosted-Nexus | Nexus credentials referenced by `settingsNexus.xml` |
| `AZURE_CREDENTIALS` | ACR | Service principal JSON for `azure/login` |
| `REGISTRY_LOGIN_SERVER`, `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` | ACR | ACR login server and credentials |

To push to GitHub Packages or GHCR, the repository's **Settings → Actions → General → Workflow permissions** must allow *Read and write*.

## Setting up self-hosted runners

The self-hosted workflows use `runs-on: Linux` or `runs-on: Windows`. These are default labels that GitHub adds when a runner is registered on that OS, so no custom labels are needed.

### Get the registration commands

1. Open the repository on GitHub and go to **Settings → Actions → Runners → New self-hosted runner**.
2. Pick the operating system (**Linux** or **Windows**) and the architecture (**x64**).
3. GitHub shows download and configure commands for the current runner version, with a registration token already filled in. The token expires after one hour.

The steps below follow those commands. Replace `<version>` and `<TOKEN>` with the values GitHub shows.

### Linux runner (Ubuntu)

1. **Install the prerequisites**, and let your user run Docker without `sudo`:

   ```bash
   sudo apt-get update
   sudo apt-get install -y git maven docker.io
   sudo usermod -aG docker $USER      # log out and back in for this to apply
   ```

2. **Download and extract the runner.** Use a normal user, not `root`; `config.sh` refuses to run as root.

   ```bash
   mkdir ~/actions-runner && cd ~/actions-runner
   curl -o actions-runner-linux-x64-<version>.tar.gz -L \
     https://github.com/actions/runner/releases/download/v<version>/actions-runner-linux-x64-<version>.tar.gz
   tar xzf ./actions-runner-linux-x64-<version>.tar.gz
   ```

3. **Register the runner** with this repository. Press Enter to accept the default runner group and labels (`self-hosted`, `Linux`, `X64`).

   ```bash
   ./config.sh --url https://github.com/charanaswanthss/githubActions --token <TOKEN> --name linux-runner-01
   ```

4. **Start the runner.** Use either option:

   ```bash
   ./run.sh                     # in the foreground, for a quick test (stop with Ctrl+C)

   sudo ./svc.sh install        # or as a systemd service that starts on boot
   sudo ./svc.sh start
   sudo ./svc.sh status
   ```

5. *(Optional, for the Nexus workflow)* run Nexus locally:

   ```bash
   docker run -d --name nexus -p 8081:8081 sonatype/nexus3
   docker exec nexus cat /nexus-data/admin.password   # initial admin password
   ```

   Then add `NEXUS_USERNAME` and `NEXUS_PASSWORD` as repository secrets.

> If you add the runner user to the `docker` group after installing the service, restart the service (`sudo ./svc.sh stop && sudo ./svc.sh start`) so it picks up the new group.

### Windows runner

Run these steps in **PowerShell as Administrator**.

1. **Install the prerequisites.** This example uses [Chocolatey](https://chocolatey.org/install):

   ```powershell
   choco install -y git maven
   ```

   Open a new PowerShell window afterwards so `git` and `mvn` are on `PATH`.

2. **Download and extract the runner.** Use a short path near the drive root, such as `C:\actions-runner`, to avoid Windows path-length and permission problems.

   ```powershell
   mkdir C:\actions-runner; cd C:\actions-runner
   Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v<version>/actions-runner-win-x64-<version>.zip -OutFile actions-runner-win-x64-<version>.zip
   Add-Type -AssemblyName System.IO.Compression.FileSystem
   [System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD\actions-runner-win-x64-<version>.zip", "$PWD")
   ```

3. **Register the runner** with this repository. When prompted *"Would you like to run the runner as service?"*, answer **Y** to install it as a Windows service that starts on boot.

   ```powershell
   .\config.cmd --url https://github.com/charanaswanthss/githubActions --token <TOKEN> --name windows-runner-01
   ```

4. **Start the runner.** If you didn't install it as a service, run it in the foreground:

   ```powershell
   .\run.cmd
   ```

   If it runs as a service, manage it with `Get-Service "actions.runner.*"` and `Restart-Service "actions.runner.*"`. Restart it after installing new tools so it picks up the updated `PATH`.

### Verify the runners

Under **Settings → Actions → Runners**, each runner should show as **Idle** with its labels. Start a self-hosted workflow from the **Actions** tab. The job log's *Set up job* step shows which runner picked it up.

### Remove a runner

Get a removal token from the runner's page under **Settings → Actions → Runners**, then run:

```bash
sudo ./svc.sh stop && sudo ./svc.sh uninstall              # Linux, if installed as a service
./config.sh remove --token <TOKEN>                         # Linux
.\config.cmd remove --token <TOKEN>                        # Windows (also removes the service)
```