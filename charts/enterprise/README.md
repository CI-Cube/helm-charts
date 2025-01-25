# CICube Enterprise Installation Guide

This guide contains the steps needed to install CICube Enterprise on your Kubernetes cluster.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- GitHub account and organization
- 3 domains (for API, Web UI, and Cube)

## 1. Create GitHub App

1. Go to your GitHub organization settings
2. Select "GitHub Apps" from the left menu
3. Click "New GitHub App" button
4. Fill in the following details:
   - **GitHub App name**: `cicube-enterprise` (or any name you prefer)
   - **Homepage URL**: Your Web UI domain (e.g., `https://app.cicube.io`)
   - **Webhook URL**: Your API domain + `/api/github/webhook` (e.g., `https://api.cicube.io/api/github/webhook`)
   - **Webhook - Active**: **Uncheck** this option
   - **Expire user authorization tokens**: **Uncheck** this option
   - **Request user authorization (OAuth) during installation**: **Check** this option

5. Set the following permissions:
   - **Repository permissions**:
     - Actions: Read
     - Metadata: Read
     - Webhooks: Read & Write
   - **Organization permissions**:
     - Members: Read
     - Plan: Read
   - **Account permissions**:
     - Email addresses: Read

6. Click "Create GitHub App" button
7. Save the following information from the created app:
   - **App ID**
   - **Client ID**
   - Click "Generate a new client secret" and save the **Client Secret**

## 2. Add Helm Repositories

```bash
helm repo add cicube https://ci-cube.github.io/helm-charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

## 3. Create Registry Secret

```bash
kubectl create secret docker-registry registry-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_GITHUB_TOKEN \
  --docker-email=YOUR_EMAIL
```

## 4. Generate Secure Passwords

```bash
# Generate PostgreSQL password (32 characters)
openssl rand -hex 16

# Generate JWT secret (32 characters)
openssl rand -hex 16
```

## 5. Create Values File

Save the following content as `values.yaml`. Replace all values marked with YOUR_* with your own values:

```yaml
global:
  imagePullSecrets:
    - name: registry-secret
  
  secrets:
    postgresql:
      password: "YOUR_POSTGRESQL_PASSWORD"  # Generated with openssl rand -hex 16
      host: ""  # Optional: Leave empty to use default
    redis:
      host: ""  # Optional: Leave empty to use default
    jwt:
      secret: "YOUR_JWT_SECRET"  # Generated with openssl rand -hex 16
    github:
      clientId: "YOUR_GITHUB_CLIENT_ID"  # From GitHub App
      clientSecret: "YOUR_GITHUB_CLIENT_SECRET"  # From GitHub App
      appName: "YOUR_GITHUB_APP_NAME"  # From GitHub App (e.g., cicube-enterprise)
    google:
      clientSecret: ""  # Optional: For Google OAuth
    smtp:
      username: ""  # Optional: For email notifications
      password: ""  # Optional: For email notifications
    loops:
      apiKey: ""  # Optional: For Loops integration

api:
  enabled: true
  replicaCount: 1
  
  image:
    repository: ghcr.io/ci-cube/cicube-api/api
    tag: "sha-f72fb74"
    pullPolicy: IfNotPresent

  ingress:
    enabled: true
    annotations:
      cert-manager.io/issuer: "letsencrypt-prod"  # Optional: For automatic TLS
    hosts:
      - host: "api.YOUR_DOMAIN.com"  # Replace with your API domain
        paths:
          - path: /
            pathType: ImplementationSpecific
    tls:
      - secretName: "api-tls"
        hosts:
          - "api.YOUR_DOMAIN.com"  # Replace with your API domain

  # API Environment Variables
  DATABASE_USER: "postgres"
  DATABASE_NAME: "halley"
  DATABASE_SSL_MODE: "false"
  REDIS_PORT: "6379"
  WEB_URL: "https://app.YOUR_DOMAIN.com"  # Replace with your Web UI domain
  API_URL: "https://api.YOUR_DOMAIN.com"  # Replace with your API domain
  SMTP_HOST: ""  # Optional: For email notifications
  SMTP_PORT: ""  # Optional: For email notifications
  SMTP_FROM: ""  # Optional: For email notifications

app:
  enabled: true
  replicaCount: 1
  
  image:
    repository: ghcr.io/ci-cube/cicube-app/app
    tag: "sha-6b59111"  # Replace with specific version
    pullPolicy: IfNotPresent

  ingress:
    enabled: true
    annotations:
      cert-manager.io/issuer: "letsencrypt-prod"  # Optional: For automatic TLS
    hosts:
      - host: "app.YOUR_DOMAIN.com"  # Replace with your Web UI domain
        paths:
          - path: /
            pathType: ImplementationSpecific
    tls:
      - secretName: "app-tls"
        hosts:
          - "app.YOUR_DOMAIN.com"  # Replace with your Web UI domain

  env:
    VITE_API_URL: "https://api.YOUR_DOMAIN.com"  # Replace with your API domain
    VITE_WEB_URL: "https://app.YOUR_DOMAIN.com"  # Replace with your Web UI domain
    VITE_CUBE_API_URL: "https://cube.YOUR_DOMAIN.com"  # Replace with your Cube domain
    VITE_GA_MEASUREMENT_ID: ""  # Optional: For Google Analytics

cubejs:
  enabled: true
  
  annotations: 
    reloader.stakater.com/auto: "true"
  
  ingress:
    enabled: true
    annotations:
      cert-manager.io/issuer: "letsencrypt-prod"  # Optional: For automatic TLS
    hosts:
      - host: "cube.YOUR_DOMAIN.com"  # Replace with your Cube domain
        paths:
          - path: /
            pathType: ImplementationSpecific
    tls:
      - secretName: "cube-tls"
        hosts:
          - "cube.YOUR_DOMAIN.com"  # Replace with your Cube domain

  cubeApi:
    image: ghcr.io/ci-cube/cicube-api/cube
    tag: "sha-8695f0c"  # Replace with specific version
    pullPolicy: IfNotPresent
    service:
      type: ClusterIP
      port: 4000
    replicas: 1

  cubeRefreshWorker:
    image: ghcr.io/ci-cube/cicube-api/cube
    tag: "sha-8695f0c"  # Replace with specific version
    pullPolicy: IfNotPresent
    replicas: 1

postgresql:
  enabled: true
  auth:
    existingSecret: "{{ .Release.Name }}-app-secrets"
    secretKeys:
      adminPasswordKey: postgresql-password
      userPasswordKey: postgresql-password
    username: "postgres"
    database: "halley"
  primary:
    persistence:
      enabled: true
      size: 10Gi  # Adjust based on your needs
```

Important notes about values.yaml:

1. **Domains**: Replace all instances of `YOUR_DOMAIN.com` with your actual domain
   - API: api.YOUR_DOMAIN.com
   - Web UI: app.YOUR_DOMAIN.com
   - Cube: cube.YOUR_DOMAIN.com

2. **Required Secrets**: These must be set
   - `global.secrets.postgresql.password`: PostgreSQL password
   - `global.secrets.jwt.secret`: JWT secret
   - `global.secrets.github.clientId`: GitHub Client ID
   - `global.secrets.github.clientSecret`: GitHub Client Secret
   - `global.secrets.github.appName`: GitHub App name

3. **Storage**: Adjust PostgreSQL storage size based on your needs
   - `postgresql.primary.persistence.size`: Default is 10Gi

## 6. Install

```bash
# Create namespace
kubectl create namespace cicube

# Install Helm chart
helm install cicube cicube/halley-enterprise \
  -f values.yaml \
  -n cicube
```

## 7. Verify Installation

```bash
# Check pod status
kubectl get pods -n cicube

# Check ingress status
kubectl get ingress -n cicube
```

The installation is complete when all pods are in Running state and ingresses are ready.

## 8. DNS Setup

Point the following domains to your Kubernetes cluster IP:
- api.domain.com
- app.domain.com
- cube.domain.com

## Troubleshooting

### If Pods Are Not Starting
```bash
# Check events
kubectl get events -n cicube

# Check pod logs
kubectl logs -f POD_NAME -n cicube
```

### Check Database Connection
```bash
# Connect to PostgreSQL pod
kubectl exec -it $(kubectl get pod -l app.kubernetes.io/name=postgresql -n cicube -o jsonpath='{.items[0].metadata.name}') -n cicube -- psql -U postgres -d halley
```

### Certificate Issues
If your TLS certificates are not ready, you can temporarily disable TLS in ingresses:
```yaml
api:
  ingress:
    enabled: true
    tls: []  # Disable TLS

app:
  ingress:
    enabled: true
    tls: []  # Disable TLS

cubejs:
  ingress:
    enabled: true
    tls: []  # Disable TLS
```

