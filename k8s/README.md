# Backstage Kubernetes Deployment

This directory contains Kubernetes manifests for deploying Backstage on a local docker-desktop cluster.

## Prerequisites

1. Docker Desktop with Kubernetes enabled
2. `kubectl` configured to use docker-desktop context
3. A Backstage Docker image (built locally and tagged as `backstage:latest`)

## Building the Backstage Image

Before deploying, you need to build a Backstage Docker image. If you haven't already:

```bash
# From your Backstage app root directory
yarn install
yarn tsc
yarn build:backend
docker image build . -f packages/backend/Dockerfile --tag backstage:latest
```

## Configuration

### GitHub Token (Required)

Update `backstage-secrets.yaml` with your GitHub Personal Access Token:

```bash
# Replace 'your-github-token-here' with your actual token
echo -n "your_actual_github_token" | base64
# Copy the output and update the GITHUB_TOKEN value in backstage-secrets.yaml
```

### PostgreSQL Credentials

The default credentials are:

- Username: `backstage`
- Password: `hunter2`

To change these, update `postgres-secrets.yaml` with new base64-encoded values:

```bash
echo -n "your_username" | base64
echo -n "your_password" | base64
```

## Deployment Steps

Deploy the manifests in this order:

```bash
# 1. Create the namespace
kubectl apply -f namespace.yaml

# 2. Create secrets
kubectl apply -f postgres-secrets.yaml
kubectl apply -f backstage-secrets.yaml

# 3. Create PostgreSQL storage
kubectl apply -f postgres-storage.yaml

# 4. Deploy PostgreSQL
kubectl apply -f postgres.yaml
kubectl apply -f postgres-service.yaml

# 5. Wait for PostgreSQL to be ready
kubectl wait --for=condition=available --timeout=300s deployment/postgres -n backstage

# 6. Deploy Backstage
kubectl apply -f backstage.yaml
```

Or deploy all at once:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f postgres-secrets.yaml
kubectl apply -f backstage-secrets.yaml
kubectl apply -f postgres-storage.yaml
kubectl apply -f postgres.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f backstage.yaml
```

## Accessing Backstage

Use port-forwarding to access Backstage locally:

```bash
kubectl port-forward --namespace=backstage svc/backstage 8080:80
```

Then open your browser to http://localhost:8080

## Verifying Deployment

Check the status of your deployments:

```bash
# View all resources in the backstage namespace
kubectl get all -n backstage

# Check pod logs
kubectl logs -n backstage deployment/backstage
kubectl logs -n backstage deployment/postgres

# Describe pods if there are issues
kubectl describe pod -n backstage
```

## Troubleshooting

### Backstage pod fails to start

1. Check if the image exists: `docker images | grep backstage`
2. Check pod logs: `kubectl logs -n backstage -l app=backstage`
3. Ensure PostgreSQL is running: `kubectl get pods -n backstage -l app=postgres`

### Database connection issues

1. Verify PostgreSQL is running: `kubectl get pods -n backstage -l app=postgres`
2. Check PostgreSQL logs: `kubectl logs -n backstage deployment/postgres`
3. Verify secrets are correct: `kubectl get secret -n backstage`

### Image pull issues

If using a local image, ensure `imagePullPolicy: IfNotPresent` is set in the deployment (it is by default).

## Cleanup

To remove all resources:

```bash
kubectl delete namespace backstage
kubectl delete pv postgres-storage
```

## Notes

- This is a development setup suitable for local testing
- For production, consider:
  - Using a managed database service
  - Implementing proper ingress/load balancing
  - Adding resource limits and requests
  - Enabling horizontal pod autoscaling
  - Using proper secret management (e.g., Sealed Secrets, external secret operators)
  - Using persistent volumes backed by network storage
