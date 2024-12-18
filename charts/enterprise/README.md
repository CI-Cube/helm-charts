# Enterprise Helm Chart

This Helm chart is used to deploy all components of the application (API, Cube, Redis, and PostgreSQL) on Kubernetes.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- Secret for Container Registry access
- Access to ghcr.io container registry

## Step by Step Installation Guide

### 1. Prepare Your Environment

1. Clone the repository and navigate to the helm chart directory:
```bash
cd k8s/enterprise
```

2. Create a namespace for your deployment (optional):
```bash
kubectl create namespace my-app
kubectl config set-context --current --namespace=my-app
```

### 2. Configure Registry Access

1. Create a Personal Access Token (PAT) in GitHub with `read:packages` permission

2. Create registry secret:
```bash
kubectl create secret docker-registry registry-secret \
  --docker-server=ghcr.io \
  --docker-username=<your-github-username> \
  --docker-password=<your-github-pat> \
  --docker-email=<your-email>
```

3. Verify the secret:
```bash
kubectl get secret registry-secret
```

### 3. Configure Dependencies

1. Add Bitnami Helm repository:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

2. Update Helm dependencies:
```bash
helm dependency update
```

### 4. Configure Values

1. Create your own values file:
```bash
cp values.yaml my-values.yaml
```

2. Edit my-values.yaml with your configuration. Required fields:
```yaml
api:
  env:
    DATABASE_USER: "your-db-user"
    DATABASE_PASSWORD: "your-db-password"
    DATABASE_NAME: "your-db-name"
    JWT_SECRET: "your-jwt-secret"
    WEB_URL: "https://your-web-url"
    API_URL: "https://your-api-url"

cube:
  env:
    CUBEJS_DB_USER: "your-db-user"
    CUBEJS_DB_PASS: "your-db-password"
    CUBEJS_DB_NAME: "your-db-name"
    CUBEJS_API_SECRET: "your-jwt-secret"

postgresql:
  auth:
    postgresPassword: "your-postgres-password"
    database: "your-db-name"

redis:
  auth:
    password: "your-redis-password"
```

### 5. Install the Chart

1. Perform a dry-run to verify your configuration:
```bash
helm install my-release . -f my-values.yaml --dry-run
```

2. Install the chart:
```bash
helm install my-release . -f my-values.yaml
```

3. Verify the deployment:
```bash
# Check all resources
kubectl get all

# Check pods status
kubectl get pods

# Check services
kubectl get svc

# Check deployments
kubectl get deployments
```

### 6. Verify the Installation

1. Wait for all pods to be ready:
```bash
kubectl wait --for=condition=ready pod --all --timeout=300s
```

2. Check API health:
```bash
# Port forward the API service
kubectl port-forward svc/my-release-api 3000:80

# In another terminal, test the API
curl http://localhost:3000/health
```

3. Check Cube health:
```bash
# Port forward the Cube service
kubectl port-forward svc/my-release-cube 4000:4000

# In another terminal, test Cube
curl http://localhost:4000/readiness
```

### 7. Troubleshooting

If you encounter issues:

1. Check pod logs:
```bash
# For API
kubectl logs -l app=my-release-api

# For Cube
kubectl logs -l app=my-release-cube

# For PostgreSQL
kubectl logs -l app=my-release-postgresql

# For Redis
kubectl logs -l app=my-release-redis
```

2. Check pod description:
```bash
kubectl describe pod <pod-name>
```

### 8. Uninstall

To remove the deployment:
```bash
helm uninstall my-release
```

## Configuration

### Global Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.imageRegistry` | Container registry address | `ghcr.io` |
| `global.imagePullSecrets` | Image pull secrets | `[{name: registry-secret}]` |

### API Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `api.image.repository` | API image repository | `ci-cube/cicube-api/api` |
| `api.image.tag` | API image tag | `latest` |
| `api.replicaCount` | Number of API replicas | `1` |

### Cube Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `cube.image.repository` | Cube image repository | `ci-cube/cicube-api/cube` |
| `cube.image.tag` | Cube image tag | `latest` |
| `cube.replicaCount` | Number of Cube replicas | `1` |

### PostgreSQL Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `postgresql.auth.postgresPassword` | PostgreSQL root password | `""` |
| `postgresql.auth.database` | Database name | `""` |

### Redis Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `redis.auth.password` | Redis password | `""` |
| `redis.architecture` | Redis architecture | `standalone` |

## Security Notes

1. All sensitive information (passwords, API keys, etc.) are left empty in the values file. You need to provide these values securely.
2. Review all security settings before using in production.
3. Use specific versions for image tags instead of `latest`.
