# 🔄 CI/CD Pipeline Documentation

## 📚 Зміст
1. [Overview](#overview)
2. [GitHub Actions Workflows](#github-actions-workflows)
3. [Testing Pipeline](#testing-pipeline)
4. [Build Pipeline](#build-pipeline)
5. [Deployment Pipeline](#deployment-pipeline)
6. [Security Scanning](#security-scanning)
7. [Monitoring](#monitoring)

## 🎯 Overview

### Pipeline Architecture

```
┌─────────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐
│   Commit    │───>│   Test   │───>│  Build  │───>│  Deploy  │
└─────────────┘    └──────────┘    └─────────┘    └──────────┘
                         │               │              │
                    ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
                    │  Lint   │    │  Docker │    │   GKE   │
                    │  Unit   │    │   Push  │    │  Update │
                    │  E2E    │    │   Scan  │    │  Verify │
                    └─────────┘    └─────────┘    └─────────┘
```

### Branch Strategy

- `main` - Production branch (auto-deploy to production)
- `develop` - Development branch (auto-deploy to staging)
- `feature/*` - Feature branches (run tests only)
- `hotfix/*` - Hotfix branches (fast-track to production)

## 🎬 GitHub Actions Workflows

### Main CI/CD Workflow

```yaml
# .github/workflows/main.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  GKE_CLUSTER: finance-cluster
  GKE_ZONE: us-central1-a
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  REGISTRY: gcr.io

jobs:
  # ============= FRONTEND TESTING =============
  test-frontend:
    name: Test Frontend
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: ./frontend
        run: npm ci

      - name: Run linter
        working-directory: ./frontend
        run: npm run lint

      - name: Run tests
        working-directory: ./frontend
        run: npm run test:ci

      - name: Build application
        working-directory: ./frontend
        run: npm run build

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./frontend/coverage/lcov.info
          flags: frontend

  # ============= BACKEND TESTING =============
  test-backend:
    name: Test Backend
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'
          cache: true
          cache-dependency-path: backend/go.sum

      - name: Install dependencies
        working-directory: ./backend
        run: go mod download

      - name: Run linter
        uses: golangci/golangci-lint-action@v3
        with:
          working-directory: ./backend
          version: latest

      - name: Run migrations
        working-directory: ./backend
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/test_db?sslmode=disable
        run: |
          go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
          migrate -path ./migrations -database "$DATABASE_URL" up

      - name: Run tests
        working-directory: ./backend
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_USER: postgres
          DB_PASSWORD: test
          DB_NAME: test_db
          REDIS_HOST: localhost
          REDIS_PORT: 6379
        run: go test -v -race -coverprofile=coverage.out ./...

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./backend/coverage.out
          flags: backend

  # ============= E2E TESTING =============
  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: [test-frontend, test-backend]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Start services with Docker Compose
        run: |
          docker-compose -f docker-compose.test.yml up -d
          sleep 10

      - name: Run E2E tests
        working-directory: ./e2e
        run: |
          npm ci
          npm run test:e2e

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: e2e-results
          path: e2e/test-results

  # ============= SECURITY SCANNING =============
  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  # ============= BUILD & PUSH =============
  build-and-push:
    name: Build and Push Docker Images
    runs-on: ubuntu-latest
    needs: [test-frontend, test-backend, e2e-tests]
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop')

    outputs:
      frontend-image: ${{ steps.image.outputs.frontend }}
      backend-image: ${{ steps.image.outputs.backend }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Configure Docker for GCR
        run: gcloud auth configure-docker

      - name: Generate image tags
        id: image
        run: |
          VERSION=${GITHUB_SHA::8}
          echo "frontend=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/finance-frontend:$VERSION" >> $GITHUB_OUTPUT
          echo "backend=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/finance-backend:$VERSION" >> $GITHUB_OUTPUT

      - name: Build and push Frontend
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: true
          tags: |
            ${{ steps.image.outputs.frontend }}
            ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/finance-frontend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_VERSION=${{ github.sha }}
            BUILD_TIME=${{ github.event.head_commit.timestamp }}

      - name: Build and push Backend
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: |
            ${{ steps.image.outputs.backend }}
            ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/finance-backend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan Docker images
        run: |
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy image ${{ steps.image.outputs.frontend }}
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy image ${{ steps.image.outputs.backend }}

  # ============= DEPLOY TO STAGING =============
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.finance.yourdomain.com

    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Get GKE credentials
        uses: google-github-actions/get-gke-credentials@v1
        with:
          cluster_name: ${{ env.GKE_CLUSTER }}
          location: ${{ env.GKE_ZONE }}

      - name: Deploy to Kubernetes
        run: |
          # Update image tags
          kubectl set image deployment/backend backend=${{ needs.build-and-push.outputs.backend-image }} \
            -n finance-dashboard-staging
          kubectl set image deployment/frontend frontend=${{ needs.build-and-push.outputs.frontend-image }} \
            -n finance-dashboard-staging

          # Wait for rollout
          kubectl rollout status deployment/backend -n finance-dashboard-staging
          kubectl rollout status deployment/frontend -n finance-dashboard-staging

      - name: Run smoke tests
        run: |
          curl -f https://staging.finance.yourdomain.com/health || exit 1
          curl -f https://staging.finance.yourdomain.com/api/health || exit 1

  # ============= DEPLOY TO PRODUCTION =============
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://finance.yourdomain.com

    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Get GKE credentials
        uses: google-github-actions/get-gke-credentials@v1
        with:
          cluster_name: ${{ env.GKE_CLUSTER }}
          location: ${{ env.GKE_ZONE }}

      - name: Create backup
        run: |
          kubectl create job backup-$(date +%s) \
            --from=cronjob/db-backup \
            -n finance-dashboard

      - name: Deploy to Kubernetes (Canary)
        run: |
          # Deploy canary version (10% traffic)
          kubectl apply -f kubernetes/canary/backend-canary.yaml
          kubectl apply -f kubernetes/canary/frontend-canary.yaml

          # Update canary images
          kubectl set image deployment/backend-canary backend=${{ needs.build-and-push.outputs.backend-image }} \
            -n finance-dashboard
          kubectl set image deployment/frontend-canary frontend=${{ needs.build-and-push.outputs.frontend-image }} \
            -n finance-dashboard

          # Wait for canary rollout
          kubectl rollout status deployment/backend-canary -n finance-dashboard
          kubectl rollout status deployment/frontend-canary -n finance-dashboard

      - name: Run canary tests
        run: |
          sleep 60
          # Check error rates and performance
          ./scripts/canary-validation.sh

      - name: Promote to production
        run: |
          # Update main deployments
          kubectl set image deployment/backend backend=${{ needs.build-and-push.outputs.backend-image }} \
            -n finance-dashboard
          kubectl set image deployment/frontend frontend=${{ needs.build-and-push.outputs.frontend-image }} \
            -n finance-dashboard

          # Wait for rollout
          kubectl rollout status deployment/backend -n finance-dashboard
          kubectl rollout status deployment/frontend -n finance-dashboard

          # Remove canary deployments
          kubectl delete deployment backend-canary frontend-canary -n finance-dashboard

      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Production deployment completed!'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        if: always()
```

## 🧪 Testing Pipeline

### Frontend Testing

```yaml
# .github/workflows/frontend-tests.yml
name: Frontend Tests

on:
  push:
    paths:
      - 'frontend/**'
      - '.github/workflows/frontend-tests.yml'

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [16, 18, 20]

    steps:
      - uses: actions/checkout@v4

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: ./frontend
        run: npm ci

      - name: Lint code
        working-directory: ./frontend
        run: npm run lint

      - name: Type check
        working-directory: ./frontend
        run: npm run type-check

      - name: Run unit tests
        working-directory: ./frontend
        run: npm run test:unit

      - name: Run integration tests
        working-directory: ./frontend
        run: npm run test:integration

      - name: Generate coverage report
        working-directory: ./frontend
        run: npm run test:coverage

      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### Backend Testing

```yaml
# .github/workflows/backend-tests.yml
name: Backend Tests

on:
  push:
    paths:
      - 'backend/**'
      - '.github/workflows/backend-tests.yml'

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        go-version: ['1.20', '1.21']

    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Setup Go ${{ matrix.go-version }}
        uses: actions/setup-go@v4
        with:
          go-version: ${{ matrix.go-version }}

      - name: Install dependencies
        working-directory: ./backend
        run: |
          go mod download
          go install gotest.tools/gotestsum@latest

      - name: Run linter
        uses: golangci/golangci-lint-action@v3
        with:
          working-directory: ./backend

      - name: Run tests with coverage
        working-directory: ./backend
        env:
          DB_HOST: localhost
          DB_PORT: 5432
        run: |
          gotestsum --format testname -- -v -race -coverprofile=coverage.out ./...
          go tool cover -html=coverage.out -o coverage.html

      - name: Benchmark tests
        working-directory: ./backend
        run: go test -bench=. -benchmem ./...

      - name: Upload coverage
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: backend/coverage.html
```

## 🏗️ Build Pipeline

### Multi-stage Docker Build

```yaml
# .github/workflows/docker-build.yml
name: Docker Build

on:
  schedule:
    - cron: '0 2 * * 1' # Weekly on Monday
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push multi-arch images
        run: |
          docker buildx build \
            --platform linux/amd64,linux/arm64 \
            --tag yourusername/finance-backend:latest \
            --push \
            ./backend

          docker buildx build \
            --platform linux/amd64,linux/arm64 \
            --tag yourusername/finance-frontend:latest \
            --push \
            ./frontend
```

## 🚀 Deployment Pipeline

### Automated Rollback

```yaml
# .github/workflows/rollback.yml
name: Rollback Deployment

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to rollback'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      version:
        description: 'Version to rollback to'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Get GKE credentials
        uses: google-github-actions/get-gke-credentials@v1
        with:
          cluster_name: ${{ env.GKE_CLUSTER }}
          location: ${{ env.GKE_ZONE }}

      - name: Rollback deployment
        run: |
          NAMESPACE="finance-dashboard"
          if [ "${{ github.event.inputs.environment }}" == "staging" ]; then
            NAMESPACE="finance-dashboard-staging"
          fi

          kubectl rollout undo deployment/backend -n $NAMESPACE
          kubectl rollout undo deployment/frontend -n $NAMESPACE

          kubectl rollout status deployment/backend -n $NAMESPACE
          kubectl rollout status deployment/frontend -n $NAMESPACE
```

## 🔐 Security Scanning

### Dependency Scanning

```yaml
# .github/workflows/dependency-scan.yml
name: Dependency Security Scan

on:
  schedule:
    - cron: '0 0 * * *' # Daily
  push:
    branches: [main, develop]

jobs:
  scan-frontend:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        working-directory: ./frontend
        run: |
          npm audit --audit-level=moderate
          npm outdated || true

      - name: Run Snyk scan
        uses: snyk/actions/node@master
        with:
          args: --severity-threshold=medium
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  scan-backend:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run Go vulnerability check
        working-directory: ./backend
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

      - name: Run Nancy scan
        working-directory: ./backend
        run: |
          go list -json -deps | docker run --rm -i sonatypecorp/nancy:latest sleuth
```

### SAST (Static Application Security Testing)

```yaml
# .github/workflows/sast.yml
name: SAST Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  codeql:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        language: ['javascript', 'go']

    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: ${{ matrix.language }}

      - name: Autobuild
        uses: github/codeql-action/autobuild@v2

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2

  semgrep:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: auto
```

## 📊 Monitoring

### Performance Monitoring

```yaml
# .github/workflows/performance.yml
name: Performance Tests

on:
  schedule:
    - cron: '0 */6 * * *' # Every 6 hours
  workflow_dispatch:

jobs:
  lighthouse:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          urls: |
            https://finance.yourdomain.com
            https://finance.yourdomain.com/dashboard
            https://finance.yourdomain.com/transactions
          uploadArtifacts: true
          temporaryPublicStorage: true
          runs: 3

      - name: Performance budget check
        run: |
          # Check if performance score is above 90
          if [ $(jq '.categories.performance.score' lighthouse-results.json) -lt 0.9 ]; then
            echo "Performance score below threshold"
            exit 1
          fi

  load-testing:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run k6 load tests
        uses: grafana/k6-action@v0.3.0
        with:
          filename: tests/load/k6-script.js
          cloud: true
        env:
          K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}
```

## 🔄 Continuous Deployment

### GitOps with ArgoCD

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: finance-dashboard
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/finance-dashboard
    targetRevision: HEAD
    path: kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: finance-dashboard
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - Validate=true
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

## 📝 Scripts

### Canary Validation Script

```bash
#!/bin/bash
# scripts/canary-validation.sh

set -e

NAMESPACE="finance-dashboard"
CANARY_WEIGHT=10
ERROR_THRESHOLD=5
LATENCY_THRESHOLD=500

echo "Starting canary validation..."

# Check error rate
ERROR_RATE=$(kubectl exec -n monitoring prometheus-0 -- \
  promtool query instant 'rate(http_requests_total{status=~"5..",deployment="backend-canary"}[5m])' | \
  jq -r '.data.result[0].value[1]')

if [ $(echo "$ERROR_RATE > $ERROR_THRESHOLD" | bc) -eq 1 ]; then
  echo "Error rate too high: $ERROR_RATE%"
  exit 1
fi

# Check latency
LATENCY=$(kubectl exec -n monitoring prometheus-0 -- \
  promtool query instant 'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{deployment="backend-canary"}[5m]))' | \
  jq -r '.data.result[0].value[1]')

if [ $(echo "$LATENCY > $LATENCY_THRESHOLD" | bc) -eq 1 ]; then
  echo "Latency too high: ${LATENCY}ms"
  exit 1
fi

echo "Canary validation passed!"
```

### Database Migration Script

```bash
#!/bin/bash
# scripts/migrate.sh

set -e

# Get database credentials from Kubernetes secret
DB_HOST=$(kubectl get secret db-secret -n finance-dashboard -o jsonpath='{.data.host}' | base64 -d)
DB_USER=$(kubectl get secret db-secret -n finance-dashboard -o jsonpath='{.data.username}' | base64 -d)
DB_PASSWORD=$(kubectl get secret db-secret -n finance-dashboard -o jsonpath='{.data.password}' | base64 -d)

# Run migrations
DATABASE_URL="postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:5432/finance_dashboard?sslmode=require"

migrate -path ./migrations -database "$DATABASE_URL" up

echo "Migrations completed successfully"
```

## 🎯 Best Practices

### 1. Branch Protection Rules

```json
{
  "protection_rules": {
    "main": {
      "required_status_checks": {
        "contexts": [
          "test-frontend",
          "test-backend",
          "security-scan"
        ]
      },
      "required_pull_request_reviews": {
        "required_approving_review_count": 2,
        "dismiss_stale_reviews": true
      },
      "enforce_admins": true,
      "required_linear_history": true,
      "allow_deletions": false
    }
  }
}
```

### 2. Secrets Management

```yaml
# Use GitHub Secrets for sensitive data
secrets:
  - GCP_PROJECT_ID
  - GCP_SA_KEY
  - DOCKERHUB_TOKEN
  - SNYK_TOKEN
  - SONAR_TOKEN
  - SLACK_WEBHOOK
  - K6_CLOUD_TOKEN

# Use Google Secret Manager in production
apiVersion: v1
kind: SecretProviderClass
metadata:
  name: app-secrets
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/PROJECT_ID/secrets/db-password/versions/latest"
        path: "db-password"
      - resourceName: "projects/PROJECT_ID/secrets/jwt-secret/versions/latest"
        path: "jwt-secret"
```

### 3. Monitoring & Alerting

```yaml
# .github/workflows/monitoring.yml
- name: Send metrics to Datadog
  env:
    DD_API_KEY: ${{ secrets.DD_API_KEY }}
  run: |
    curl -X POST "https://api.datadoghq.com/api/v1/series" \
      -H "DD-API-KEY: ${DD_API_KEY}" \
      -d @- << EOF
    {
      "series": [{
        "metric": "deployment.success",
        "points": [[$(date +%s), 1]],
        "tags": ["env:${{ github.event.inputs.environment }}"]
      }]
    }
    EOF
```

## 🚨 Troubleshooting

### Common Issues

1. **Build Failures**
```bash
# Check build logs
gh run view <run-id> --log

# Re-run failed jobs
gh run rerun <run-id> --failed
```

2. **Deployment Failures**
```bash
# Check deployment status
kubectl describe deployment <name> -n finance-dashboard

# View pod logs
kubectl logs deployment/<name> -n finance-dashboard --tail=100
```

3. **Secret Issues**
```bash
# Verify secrets are set
gh secret list

# Update secret
gh secret set SECRET_NAME
```