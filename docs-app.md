# ShopVerse Documentation

## Project Workflow

The ShopVerse application follows a modern CI/CD pipeline with the following workflow:

1. **Development**: Code changes are made in `frontend/` (React) or `backend/` (Go) directories
2. **Push to main**: When changes are pushed to the `main` branch, GitHub Actions triggers
3. **Security Scan**: Trivy scans the codebase for vulnerabilities
4. **Build & Push**: Docker images are built using multi-stage builds and pushed to Amazon ECR
5. **Deployment**: The deploy workflow updates Helm chart image tags and syncs ArgoCD to deploy to Kubernetes

## Dockerfiles

### frontend/Dockerfile

Multi-stage build for the React frontend application:

- **Stage 1 (builder)**: Uses `node:18-alpine` to install dependencies and build the static assets
- **Stage 2**: Uses `nginx:alpine` to serve the built static files

**Key steps**:

- Copies `package.json` and `package-lock.json` to install dependencies with `npm install`
- Copies the rest of the application code and builds with `npm run build`
- The `VITE_API_URL` build argument configures the API endpoint at build time
- Nginx configuration includes:
  - SPA routing via `try_files` for client-side routing
  - API proxy at `/api/` forwarding to the backend service
  - `/health` endpoint returning `ok` for health checks
  - Gzip compression for text-based assets
  - Long-term caching for static assets (JS, CSS, images, fonts)

### backend/Dockerfile

Multi-stage build for the Go backend API server:

- **Stage 1 (builder)**: Uses `golang:1.24-alpine` with GCC and Musl for CGO support
- **Stage 2**: Uses `gcr.io/distroless/static-debian12` for a minimal, secure production image

**Key steps**:

- Installs build dependencies (`gcc`, `musl-dev`) required for CGO
- Downloads Go modules using `go mod download`
- Builds a statically-linked binary with `-ldflags="-w -s"` for smaller size
- The binary is built from `cmd/main.go`
- Runs as non-root user (`nonroot:nonroot`) for security
- Exposes port 8080

## Docker Compose

`docker-compose.yml` orchestrates three services for local development:

### mysql

- Uses MySQL 8.0 official image
- Pre-configured database named `shopverse` with user `shopverse` / password `shopverse123`
- Root password: `rootpassword`
- Persists data in a named volume `mysql_data`
- Health check uses `mysqladmin ping` to verify readiness before dependent services start

### backend

- Built from `backend/Dockerfile`
- Maps internal port 8080 to host port 11000
- Environment variables for database connection and JWT secret
- Depends on `mysql` service, waits for healthy status before starting
- Configured with `restart: unless-stopped`

### frontend

- Built from `frontend/Dockerfile` with `VITE_API_URL` set to `http://localhost:8080`
- Maps internal port 80 to host port 11001
- Depends on `backend` service
- Configured with `restart: unless-stopped`

## CI/CD Pipelines

### build-frontend.yml

Triggers on push to `main` branch when files in `frontend/` or the workflow itself change.

**Steps**:

1. Checkout code
2. Trivy filesystem security scan (fails on CRITICAL vulnerabilities)
3. Configure AWS credentials via OIDC role assumption
4. Authenticate with Amazon ECR
5. Build and push Docker image to ECR with SHA tag
6. Uses GitHub Actions cache for faster subsequent builds

### build-backend.yml

Triggers on push to `main` branch when files in `backend/` or the workflow itself change.

**Steps**:

1. Checkout code
2. Trivy filesystem security scan (fails on CRITICAL vulnerabilities)
3. Configure AWS credentials via OIDC role assumption
4. Authenticate with Amazon ECR
5. Build and push Docker image to ECR with SHA tag
6. Uses GitHub Actions cache for faster subsequent builds

### deploy.yml

Triggers when both `Build Backend` and `Build Frontend` workflows complete successfully.

**Steps**:

1. Checkout the `shopverse-k8s` repository (separate infrastructure repo)
2. Setup `yq` for YAML manipulation

> yq is a YAML processor that allows you to query, modify, and update YAML files from the command line, similar to how jq works for JSON. It's used in the deploy pipeline to programmatically update the image tags in the Helm values.yaml file without manually editing the file.

1. Detect which component triggered the workflow and get the commit SHA
2. Update the corresponding image tag in `helm/shopverse/values.yaml`
3. Commit and push changes to the k8s repository
4. Install ArgoCD CLI
5. Sync the ArgoCD application to deploy the new images to Kubernetes

**Key features**:

- Concurrency control prevents overlapping deployments
- Only syncs if ArgoCD credentials are configured (falls back to auto-sync)
- Automatically determines which component(s) changed to minimize unnecessary updates

