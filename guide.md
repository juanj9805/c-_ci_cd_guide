# CI/CD Guide: .NET MVC → Docker → GitHub Actions → VPS

## Overview

This guide sets up a full CI/CD pipeline:

```
git push → GitHub Actions builds Docker image → pushes to GHCR → deploys to VPS via SSH
```

| Component | Role |
|-----------|------|
| **GHCR** (GitHub Container Registry) | Docker image registry hosted by GitHub — free and integrated with your repo |
| **GitHub Actions** | Automated workflow triggered on `git push` |
| **VPS** | Your Linux server where the container runs |

---

## Prerequisites

- .NET 10 SDK installed locally
- Docker Desktop running
- GitHub account
- A VPS with SSH access
- Basic terminal familiarity

---

## Step 1: Create the Project

```bash
dotnet new mvc -n cicd
cd cicd
rider .
```

---

## Step 2: Add the Dockerfile

Create a `Dockerfile` in the project root.

![alt text](image.png)

![alt text](image-1.png)

### Why a multi-stage build?

A single-stage Docker build includes the entire .NET SDK (~700 MB) in the final image. Multi-stage builds fix this by splitting work into two stages:

- **Stage 1 (build):** Uses the full SDK to compile and publish the app.
- **Stage 2 (runtime):** Uses only the lightweight ASP.NET runtime (~200 MB) and copies the compiled output from Stage 1.

Result: a smaller, faster, more secure production image that ships no build tooling.

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /app

# Restore dependencies first — improves layer caching on subsequent builds
COPY *.csproj ./
RUN dotnet restore

# Copy source and publish
COPY . ./
RUN dotnet publish -c Release -o /out

# Stage 2: Runtime only
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app

COPY --from=build /out .

EXPOSE 8080

# Replace "cicd" with your actual project name
ENTRYPOINT ["dotnet", "cicd.dll"]
```

**The DLL name in `ENTRYPOINT` must match your project name exactly.**

![alt text](image-2.png)

> **Port note:** ASP.NET Core 8+ listens on port **8080** by default (not 80). Use `EXPOSE 8080`.

---

## Step 3: Push the Project to GitHub

Create a new repository on GitHub and copy the remote URL:

![alt text](image-3.png)

![alt text](image-4.png)

![alt text](image-5.png)

Add it as the remote origin in your project:

![alt text](image-6.png)

```bash
git add .
git commit -m "chore: add dockerfile"
git push
```

---

## Step 4: Configure the GitHub Actions Workflow

Go to **Actions** in your repository:

![alt text](image-9.png)

GitHub may suggest a default workflow. Delete the generated content:

![alt text](image-10.png)

Replace it with the following:

```yaml
name: production

on:
  push:
    branches:
      - main

permissions:
  packages: write
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Cache Docker layers
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        run: |
          docker build -f Dockerfile -t ghcr.io/${{ github.repository }}:latest .
          docker push ghcr.io/${{ github.repository }}:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to VPS
        uses: appleboy/ssh-action@v0.1.9
        with:
          host: ${{ secrets.VPS_HOST }}
          username: root
          key: ${{ secrets.SSH_KEY }}
          port: 22
          script: |
            docker login ghcr.io -u ${{ github.actor }} -p ${{ secrets.TOKEN }}
            docker pull ghcr.io/${{ github.repository }}:latest
            docker stop ${{ github.event.repository.name }} || true
            docker rm ${{ github.event.repository.name }} || true
            docker run -d --name ${{ github.event.repository.name }} \
              -p 8080:8080 \
              ghcr.io/${{ github.repository }}:latest
```

### What each job does

| Job | Purpose |
|-----|---------|
| `build-and-push` | Compiles the Docker image and pushes it to GHCR |
| `deploy` | SSHs into the VPS, pulls the new image, and restarts the container |

### Why Buildx and layer caching?

**Buildx** is Docker's extended build toolkit. It enables BuildKit optimizations and multi-platform builds.

**Layer caching** avoids rebuilding unchanged layers on every push. The `COPY *.csproj && dotnet restore` layer only changes when you modify dependencies — so it gets cached and skipped on repeated runs, cutting build time significantly.

### Save the file — use only lowercase for the filename

Uppercase letters in workflow filenames cause failures on Linux runners.

![alt text](image-12.png)

![alt text](image-13.png)

The workflow file is now created:

![alt text](image-14.png)

![alt text](image-15.png)

Click the workflow and verify the CI job passes before moving to deployment:

![alt text](image-16.png)

![alt text](image-17.png)

---

## Step 5: Configure SSH Keys for Deployment

The `deploy` job connects to your VPS over SSH. GitHub Actions authenticates using a private key stored as a secret — no password involved.

### Why a dedicated deploy key pair?

Do not reuse your personal SSH key. A deploy key is scoped only to automation. If it gets compromised, you rotate just that key without affecting your personal access.

On your local machine, generate the key pair:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ./deploy_key
```

This creates two files:

| File | Purpose |
|------|---------|
| `deploy_key` | **Private key** — goes into GitHub Secrets as `SSH_KEY` |
| `deploy_key.pub` | **Public key** — goes onto the VPS |

![alt text](image-21.png)

### Add the public key to the VPS

SSH into your VPS:

```bash
ssh root@your_vps_ip
```

![alt text](image-22.png)

Ensure the `.ssh` directory exists and check existing keys:

```bash
mkdir -p ~/.ssh
cat ~/.ssh/authorized_keys
```

![alt text](image-23.png)

If the file is empty or missing, add the public key:

```bash
vim ~/.ssh/authorized_keys
```

Paste the full contents of `deploy_key.pub` and save.

---

## Step 6: Add GitHub Secrets

Three secrets must be set before the workflow can succeed. All names are **case-sensitive** and must match the YAML exactly.

| Secret name | Value |
|------------|-------|
| `SSH_KEY` | Full contents of the `deploy_key` private key file |
| `TOKEN` | A GitHub Personal Access Token (Classic) |
| `VPS_HOST` | Your VPS IP address |

### Add `SSH_KEY`

On Windows, open the private key file in Notepad:

![alt text](image-26.png)

![alt text](image-25.png)

Go to **Repository → Settings → Secrets and variables → Actions → New repository secret**:

![alt text](image-27.png)

![alt text](image-24.png)

Name it `SSH_KEY` and paste the private key contents:

![alt text](image-28.png)

Verify the name matches what is in the YAML:

![alt text](image-29.png)

### Create a Personal Access Token (Classic)

Go to **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**:

![alt text](image-30.png)
![alt text](image-31.png)
![alt text](image-32.png)
![alt text](image-33.png)

Select at minimum these scopes: `write:packages` and `read:packages`. These are the only permissions needed to push and pull from GHCR — do not select all scopes.

![alt text](image-35.png)

Generate the token:

![alt text](image-36.png)

**Copy the token immediately** — GitHub does not show it again.

![alt text](image-37.png)

### Add `TOKEN` secret

Back in repository **Settings → Secrets → Actions**, create a new secret named `TOKEN`:

![alt text](image-38.png)
![alt text](image-39.png)
![alt text](image-40.png)
![alt text](image-41.png)
![alt text](image-42.png)

---

## Step 7: Port Configuration on the VPS

The workflow runs the container with `-p 8080:8080`. If something else is already using port 8080 on your VPS (e.g., nginx), the container will fail to start.

![alt text](image-43.png)

Check what is listening on the port:

```bash
ss -tlnp | grep 8080
```

If it is occupied, change the **host-side** port (left of `:`):

```yaml
# In the deploy script, change 8080:8080 to something free, e.g.:
-p 8090:8080
```

The right side (`:8080`) must stay as-is — that is the port the container itself listens on.

![alt text](image-44.png)

![alt text](image-45.png)

Commit and push:

```bash
git add .
git commit -m "ci: configure deploy workflow and secrets"
git push
```

![alt text](image-46.png)

---

## Step 8: Verify the Pipeline

Monitor the workflow run under **Actions**:

![alt text](image-47.png)

### Common failure: hardcoded GitHub username

If you followed an older tutorial, the deploy script may have a hardcoded GitHub username in the `docker login` line. Replace it with `${{ github.actor }}` — which is already correct in the workflow above.

![alt text](image-48.png)

---

## Troubleshooting

| Error | Likely cause | Fix |
|-------|-------------|-----|
| SSH authentication fails | Wrong key or trailing newline in secret | Re-paste the private key, verify no extra whitespace |
| `TOKEN` auth fails | Wrong scope or expired token | Regenerate with `write:packages` scope |
| Container not reachable | Port conflict on VPS | Run `ss -tlnp \| grep 8080`, choose a different host port |
| Image not found on pull | Package visibility is private | Make the package public in GitHub Packages settings, or verify token has `read:packages` |
| Workflow not triggering | Branch name mismatch | Confirm you are pushing to `main`, not `master` |
