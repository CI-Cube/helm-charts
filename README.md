# CI-Cube Helm Charts

This repository contains Helm charts for CI-Cube applications.

## Usage

Add the Helm repository:

```bash
helm repo add cicube https://ci-cube.github.io/helm-charts
helm repo update
```

## Available Charts

### Enterprise Chart

Complete application stack including API, Cube, Redis, and PostgreSQL.

```bash
# First time installation
helm install my-release cicube/enterprise -f values.yaml

# Upgrading an existing installation
helm upgrade my-release cicube/enterprise -f values.yaml
```

For detailed configuration options, see the [enterprise chart documentation](charts/enterprise/README.md).

## Development

### Repository Structure

```
helm-charts/
├── charts/
│   └── enterprise/    # Enterprise chart
├── .github/workflows/ # CI/CD pipelines
├── cr.yaml           # Chart Releaser config
└── README.md
```

### Testing Changes Locally

```bash
# Package the chart
helm package charts/enterprise

# Create test values file
cp charts/enterprise/values.yaml my-values.yaml
# Edit my-values.yaml with your configuration

# Test installation
helm install test-release ./enterprise-0.1.0.tgz -f my-values.yaml --dry-run
```

### Release Process

Charts are automatically released using GitHub Actions when changes are pushed to the `main` branch. The process:

1. Updates chart dependencies
2. Packages the chart
3. Creates a new GitHub release
4. Updates the Helm repository index

## License

MIT
