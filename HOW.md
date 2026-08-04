# CI/CD Pipeline with GitHub Actions, Docker & GHCR

## 1. Create Two GitHub Repositories

Create two repositories for this project:

- `ci`
- `cd`

> **Important:** Keep repository names in lowercase. Docker and GitHub Container Registry (GHCR) image names are conventionally lowercase and this helps avoid image naming issues.

---

## 2. Create a Classic Personal Access Token (PAT)

Create a **Classic Personal Access Token (PAT)** (not the fine-grained token).

Navigate to:

```text
GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (Classic) → Generate New Token (Classic)
```

Grant the following permissions:

### Repository Permissions

```text
repo
```

This provides full repository access.

### Package Permissions

```text
read:packages
write:packages
delete:packages (optional)
```

Copy the generated token immediately and store it securely.

---

## 3. Create Repository Secrets

In both repositories (`ci` and `cd`):

```text
Settings → Secrets and Variables → Actions → New Repository Secret
```

Create the following secret:

```text
Name: CD_PAT
Value: <your_classic_pat_token>
```

---

# Repository 1: CI

## Create Workflow Directory

```bash
mkdir -p .github/workflows
```

---

## Create `.github/workflows/ci.yml`

```yaml
name: CI - Build, Tag and Push to GHCR

on:
  push:
    tags:
      - 'v*' # Triggers whenever you push a tag like v1.0.0, v1.1.2, etc.

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    permissions:
      contents: write
      packages: write

    steps:
      - name: 1. Checkout Latest Code
        uses: actions/checkout@v4

      - name: 2. Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.CD_PAT }}

      - name: 3. Extract Version from Git Tag
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest
            type=ref,event=tag
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: 4. Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

      - name: 5. Update CD Repository (Repo 2 - Condinum) with Version Tag
        env:
          CD_REPO_PAT: ${{ secrets.CD_PAT }}
        run: |
          # Extracts the exact version tag pushed (e.g., v1.0.0) from GitHub ref
          NEW_TAG=${GITHUB_REF#refs/tags/}
          echo "New release version tag generated: $NEW_TAG"
          
          # Configures Git identity for the runner
          git config --global user.name "GitHub Actions CI Bot"
          git config --global user.email "actions@github.com"
          
          # Clone Repo 2 (Condinum)
          git clone https://x-access-token:${{ secrets.CD_PAT }}@github.com/<username>/cd.git cd-repo
          
          cd cd-repo
          
          # Updates the image tag line in docker-compose.yml with the clean version
          LOWER_IMAGE_NAME=$(echo "${{ env.IMAGE_NAME }}" | tr '[:upper:]' '[:lower:]')
          sed -i "s|image: ghcr.io/.*|image: ghcr.io/$LOWER_IMAGE_NAME:$NEW_TAG|g" docker-compose.yml
          
          # Commits and push back to Repo 2 if changes exist
          if git diff --quiet docker-compose.yml; then
            echo "No changes detected in docker-compose.yml"
          else
            git add docker-compose.yml
            git commit -m "chore(cd): update crud-api image tag to $NEW_TAG"
            git push origin main
            echo "Successfully pushed version tag update to Condinum repo!"
          fi
```

---

## Do Not Push Yet

Do **not** push the CI repository yet.

Since the CI workflow attempts to update the CD repository, the CD repository must be created and pushed first to avoid workflow failures.

---

# Repository 2: CD

## Create `docker-compose.yml`

```yaml
version: '3.8'

services:
  crud-api:
    image: ghcr.io/<username>/<repo>:<release version>
    build: .
    ports:
      - "8080:8080"
    volumes:
      - crud-data:/app/data
    restart: always
    environment:
      - SPRING_DATASOURCE_URL=jdbc:h2:mem:cruddb;DB_CLOSE_DELAY=-1
      - SPRING_DATASOURCE_USERNAME=SA
      - SPRING_DATASOURCE_PASSWORD=

volumes:
  crud-data:
```

Replace:

- `<username>` with your GitHub username
- `<repo_name>` with your CI repository name

Example:

```yaml
image: ghcr.io/tamizhazhagan-sk/ci:latest
```

---

## Create Workflow Directory

```bash
mkdir -p .github/workflows
```

---

## Create `.github/workflows/cd.yml`

```yaml
name: CD - Deploy with Docker Compose

on:
  push:
    branches:
      - "main"
    paths:
      - 'docker-compose.yml'

  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: 1. Checkout Repository
        uses: actions/checkout@v4

      - name: 2. Log in to GitHub Container Registry (GHCR)
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.CD_PAT }}

      - name: 3. Pull and Deploy via Docker Compose with Retry
        run: |
          echo "Attempting to pull container image..."

          # Retry loop for GHCR propagation delay
          n=0
          until [ $n -ge 5 ]
          do
            docker compose pull && break
            n=$((n+1))
            echo "Manifest not ready yet (Attempt $n/5). Retrying in 15 seconds..."
            sleep 15
          done

          if [ $n -eq 5 ]; then
            echo "Error: Docker pull failed after 5 attempts."
            exit 1
          fi

          echo "Restarting containers..."
          docker compose up -d --remove-orphans

          echo "Deployment completed successfully!"
          docker compose ps

      - name: 4. Run Integration Smoke Tests
        run: |
          echo "Waiting for container application to initialize..."

          n=0
          until [ $n -ge 10 ] || curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health | grep -q "200"; do
            n=$((n+1))
            echo "App starting up (Attempt $n/10)..."
            sleep 5
          done

          echo "Testing health endpoint..."

          response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health)

          if [ "$response" -ne 200 ]; then
            echo "Error: API health check failed with status $response"
            docker compose logs
            exit 1
          fi

          echo "Success: API is responding properly!"
```

---

# Push Order (Important)

To keep the pipeline error-free:

## Step 1: Push Repository 2 (CD)

```bash
git add .
git commit -m "feat: add docker compose deployment workflow"
git branch -M main
git push origin main
```

---

## Step 2: Push Repository 1 (CI)

```bash
git add .
git commit -m "feat: add CI workflow for GHCR publishing"
git branch -M main
git push origin main
```

---

# Create First Release Tag

After pushing the CI repository:

```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0
```

This tag triggers the CI workflow automatically.

---

# End-to-End Flow

```text
Developer Creates Tag (v1.0.0)
              │
              ▼
     CI Repository Workflow
              │
              ▼
      Build Docker Image
              │
              ▼
        Push to GHCR
              │
              ▼
 Update docker-compose.yml in CD Repo
              │
              ▼
     Push Changes to CD Repo
              │
              ▼
     CD Workflow Triggered
              │
              ▼
      Docker Compose Pull
              │
              ▼
       Deploy Container
              │
              ▼
      Run Health Checks
              │
              ▼
     Deployment Successful!!!
```

## Result

You now have a complete CI/CD pipeline using:

- **GitHub Actions**
- **Docker**
- **GitHub Container Registry (GHCR)**
- **Docker Compose**
- **Classic GitHub PAT**
- **Automatic Image Version Promotion**
- **Automatic Deployment Triggering**
- **Smoke Test Validation**
- **Tag-Based Release Management**