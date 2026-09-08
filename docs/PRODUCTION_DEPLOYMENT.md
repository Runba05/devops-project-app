# Production Deployment Guide - Secure Event Ticketing Platform

This guide covers deployment of the Secure Event Ticketing Platform to Kubernetes/OpenShift in production.

## Table of Contents

1. [Pre-Deployment Requirements](#pre-deployment-requirements)
2. [Image Build and Registry Push](#image-build-and-registry-push)
3. [Security Scanning](#security-scanning)
4. [Kubernetes Deployment](#kubernetes-deployment)
5. [OpenShift Deployment](#openshift-deployment)
6. [Verification](#verification)
7. [Rolling Updates](#rolling-updates)
8. [Rollback Procedures](#rollback-procedures)
9. [Troubleshooting](#troubleshooting)
10. [Incident Response Runbook](#incident-response-runbook)

## Pre-Deployment Requirements

### Environment

- Kubernetes 1.24+ or OpenShift 4.12+
- `kubectl` or `oc` CLI installed
- Docker CLI for image building
- Access to container registry (Docker Hub, ECR, GCR, Quay, etc.)
- Helm 3.x+ (optional, for templating)

### Cluster Configuration

```bash
# Verify cluster access
kubectl cluster-info
kubectl get nodes

# Create namespace (or skip if using existing)
kubectl create namespace ticketing
```

### Required Secrets

Before deployment, prepare credentials:

```bash
# Create Docker registry secret (if private registry)
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=youruser \
  --docker-password=yourtoken \
  --docker-email=you@example.com \
  -n ticketing

# Verify secret
kubectl get secrets -n ticketing
```

### Ingress Controller

Ensure an ingress controller is installed:

```bash
# For nginx ingress controller
kubectl get deployment -n ingress-nginx

# If not installed, install it:
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

## Image Build and Registry Push

### Step 1: Build Docker Images

```bash
# Build all images locally
docker compose build

# Tag images for registry
docker tag default-api:latest registry.example.com/ticketing/api:1.0.0
docker tag default-frontend:latest registry.example.com/ticketing/frontend:1.0.0
docker tag default-worker:latest registry.example.com/ticketing/worker:1.0.0

# Or use buildx for multi-architecture (arm64, amd64)
docker buildx build --platform linux/amd64,linux/arm64 -t registry.example.com/ticketing/api:1.0.0 ./api --push
docker buildx build --platform linux/amd64,linux/arm64 -t registry.example.com/ticketing/frontend:1.0.0 ./frontend --push
docker buildx build --platform linux/amd64,linux/arm64 -t registry.example.com/ticketing/worker:1.0.0 ./worker --push
```

### Step 2: Push to Registry

```bash
# Login to registry
docker login registry.example.com

# Push images
docker push registry.example.com/ticketing/api:1.0.0
docker push registry.example.com/ticketing/frontend:1.0.0
docker push registry.example.com/ticketing/worker:1.0.0

# Verify push
docker images registry.example.com/ticketing/*
```

## Security Scanning

### Using Trivy

```bash
# Install Trivy
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
apt-get update
apt-get install trivy

# Scan images locally
trivy image localhost:5000/ticketing/api:1.0.0
trivy image localhost:5000/ticketing/frontend:1.0.0
trivy image localhost:5000/ticketing/worker:1.0.0

# Generate report
trivy image --format json --output api-scan.json registry.example.com/ticketing/api:1.0.0

# Exit with error if vulnerabilities found
trivy image --exit-code 1 --severity HIGH,CRITICAL registry.example.com/ticketing/api:1.0.0
```

### Remediation

If vulnerabilities are found:

1. **Update base images:**
   ```bash
   # In Dockerfile
   FROM node:20-alpine  # Update to latest patch version
   ```

2. **Update dependencies:**
   ```bash
   npm audit
   npm audit fix
   npm audit fix --force  # Use with caution
   ```

3. **Rebuild and rescan:**
   ```bash
   docker compose build --no-cache
   trivy image localhost:5000/ticketing/api:latest
   ```

## Kubernetes Deployment

### Step 1: Prepare Manifests

Update image references in `k8s/base/deployment.yaml`:

```yaml
# Change from latest to specific version
image: registry.example.com/ticketing/api:1.0.0
image: registry.example.com/ticketing/frontend:1.0.0
image: registry.example.com/ticketing/worker:1.0.0

# If using private registry, add imagePullSecrets
imagePullSecrets:
- name: regcred
```

### Step 2: Configure Credentials

Edit `k8s/base/deployment.yaml` and update the Secret:

```bash
# Generate new credentials
kubectl create secret generic ticketing-credentials \
  --from-literal=POSTGRES_USER=prod_user \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -base64 32)" \
  -n ticketing --dry-run=client -o yaml > /tmp/secret.yaml

# Review and apply
cat /tmp/secret.yaml
kubectl apply -f /tmp/secret.yaml
```

### Step 3: Deploy to Kubernetes

```bash
# Apply manifests in order
kubectl apply -f k8s/base/deployment.yaml
kubectl apply -f k8s/base/rbac.yaml
kubectl apply -f k8s/base/ingress-and-netpolicy.yaml

# Verify deployment
kubectl get all -n ticketing
kubectl get pvc -n ticketing
```

### Step 4: Wait for Ready State

```bash
# Monitor rollout
kubectl rollout status deployment/postgres -n ticketing
kubectl rollout status deployment/redis -n ticketing
kubectl rollout status deployment/api -n ticketing
kubectl rollout status deployment/frontend -n ticketing
kubectl rollout status deployment/worker -n ticketing
```

## OpenShift Deployment

### Step 1: Create OpenShift Project

```bash
oc new-project ticketing

# Or use existing project
oc project ticketing
```

### Step 2: Convert Ingress to Route

Replace the Ingress in `k8s/base/ingress-and-netpolicy.yaml` with OpenShift Route:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: frontend-route
  namespace: ticketing
spec:
  to:
    kind: Service
    name: frontend
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect

---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: api-route
  namespace: ticketing
spec:
  to:
    kind: Service
    name: api
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

### Step 3: Deploy

```bash
# Apply manifests
oc apply -f k8s/base/deployment.yaml
oc apply -f k8s/base/rbac.yaml
oc apply -f k8s/base/openshift-routes.yaml

# Monitor
oc get all
oc describe route frontend-route
```

## Verification

### Check Pod Status

```bash
# All pods should be Running and Ready
kubectl get pods -n ticketing

# Check pod details
kubectl describe pod -l app=api -n ticketing
kubectl logs -l app=api -n ticketing --tail=50
```

### Test API Endpoints

```bash
# Get API pod IP
API_POD=$(kubectl get pod -l app=api -o jsonpath='{.items[0].metadata.name}' -n ticketing)

# Forward port
kubectl port-forward pod/$API_POD 8080:8080 -n ticketing &

# Test endpoints
curl http://localhost:8080/healthz
curl http://localhost:8080/events
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"test@example.com","quantity":1}'
curl http://localhost:8080/tickets/orders

# Stop port-forward
pkill -f "port-forward"
```

### Check Persistent Volumes

```bash
# List PVCs
kubectl get pvc -n ticketing

# Check PostgreSQL data
kubectl exec -it deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "SELECT COUNT(*) FROM ticket_orders;"
```

## Rolling Updates

### Update Image

```bash
# Edit deployment to use new image tag
kubectl set image deployment/api api=registry.example.com/ticketing/api:2.0.0 -n ticketing

# Monitor rollout
kubectl rollout status deployment/api -n ticketing

# Check history
kubectl rollout history deployment/api -n ticketing
```

### Update Environment Variables

```bash
# Update ConfigMap
kubectl patch configmap ticketing-config -n ticketing \
  -p '{"data":{"API_BASE_URL":"http://api-v2:8080"}}'

# Restart pods to pick up changes
kubectl rollout restart deployment/api -n ticketing
kubectl rollout status deployment/api -n ticketing
```

## Rollback Procedures

### Rollback Deployment

```bash
# Check rollout history
kubectl rollout history deployment/api -n ticketing

# Rollback to previous revision
kubectl rollout undo deployment/api -n ticketing

# Rollback to specific revision
kubectl rollout undo deployment/api --to-revision=2 -n ticketing

# Verify rollback
kubectl rollout status deployment/api -n ticketing
```

### Verify Data Integrity After Rollback

```bash
# Connect to PostgreSQL
kubectl exec -it deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing

# Query orders
SELECT * FROM ticket_orders ORDER BY created_at DESC LIMIT 10;
```

## Troubleshooting

### Pod Stuck in Pending

```bash
# Check node resources
kubectl top nodes
kubectl describe nodes

# Check PVC binding
kubectl describe pvc postgres-pvc -n ticketing

# Check events
kubectl get events -n ticketing --sort-by='.lastTimestamp'
```

### Pod Crash Loop

```bash
# Check logs
kubectl logs -f deployment/api -n ticketing

# Describe pod
kubectl describe pod -l app=api -n ticketing

# Check resource limits
kubectl get resourcequota -n ticketing
```

### Database Connection Errors

```bash
# Test connectivity from API pod
kubectl exec deployment/api -n ticketing -- \
  sh -c 'nc -zv postgres 5432'

# Check DNS resolution
kubectl exec deployment/api -n ticketing -- \
  nslookup postgres

# Verify credentials
kubectl get secret ticketing-credentials -n ticketing -o yaml
```

### Redis Connection Errors

```bash
# Test connectivity
kubectl exec deployment/api -n ticketing -- \
  sh -c 'nc -zv redis 6379'

# Check Redis logs
kubectl logs deployment/redis -n ticketing
```

### Network Policy Issues

```bash
# Check if network policies are enforced
kubectl get networkpolicies -n ticketing

# Temporarily disable for debugging
kubectl delete networkpolicies -n ticketing

# Re-apply after debugging
kubectl apply -f k8s/base/ingress-and-netpolicy.yaml
```

## Incident Response Runbook

### Scenario 1: API Service Unavailable

**Symptoms:** Frontend can't reach API, requests timeout

**Diagnosis:**
```bash
# 1. Check pod status
kubectl get pods -l app=api -n ticketing

# 2. Check logs
kubectl logs -l app=api -n ticketing --tail=100

# 3. Check service
kubectl get svc api -n ticketing

# 4. Test connectivity
kubectl exec deployment/frontend -n ticketing -- curl http://api:8080/healthz
```

**Resolution:**
```bash
# Restart API deployment
kubectl rollout restart deployment/api -n ticketing

# Or rollback if recently updated
kubectl rollout undo deployment/api -n ticketing

# Monitor
kubectl rollout status deployment/api -n ticketing
```

### Scenario 2: Database Full / Out of Space

**Symptoms:** Orders not being processed, database errors

**Diagnosis:**
```bash
# Check PVC usage
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "SELECT pg_size_pretty(pg_database_size('ticketing'));"

# Check available space
kubectl exec deployment/postgres -n ticketing -- df -h /var/lib/postgresql/data
```

**Resolution:**
```bash
# Expand PVC
kubectl patch pvc postgres-pvc -n ticketing \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Or delete old data
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "DELETE FROM ticket_orders WHERE created_at < NOW() - INTERVAL '90 days';"
```

### Scenario 3: High Memory Usage

**Symptoms:** Pods OOMKilled (exit code 137)

**Diagnosis:**
```bash
# Check resource usage
kubectl top pods -n ticketing

# Check limits
kubectl get pods -o json -n ticketing | jq '.items[].spec.containers[].resources'
```

**Resolution:**
```bash
# Increase memory limits in deployment.yaml
kubectl set resources deployment/api \
  --limits=memory=1Gi,cpu=2 \
  -n ticketing

# Restart pods
kubectl rollout restart deployment/api -n ticketing
```

### Scenario 4: Queue Backlog (Orders Not Processing)

**Symptoms:** Redis queue growing, orders stuck in "queued" state

**Diagnosis:**
```bash
# Check Redis queue size
kubectl exec deployment/redis -n ticketing -- \
  redis-cli LLEN ticket_orders

# Check worker logs
kubectl logs -l app=worker -n ticketing --tail=50

# Check database for processed orders
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "SELECT COUNT(*) FROM ticket_orders WHERE status='processed';"
```

**Resolution:**
```bash
# Scale up worker replicas
kubectl scale deployment worker --replicas=5 -n ticketing

# Monitor processing
watch kubectl exec deployment/redis -n ticketing -- redis-cli LLEN ticket_orders

# Scale back down once backlog cleared
kubectl scale deployment worker --replicas=2 -n ticketing
```

### Scenario 5: Failed Rolling Update

**Symptoms:** New deployment stuck, old pods still serving traffic

**Diagnosis:**
```bash
# Check rollout status
kubectl rollout status deployment/api -n ticketing

# Check pod events
kubectl describe pod -l app=api -n ticketing

# Check image availability
kubectl get events -n ticketing | grep -i pull
```

**Resolution:**
```bash
# Immediately rollback
kubectl rollout undo deployment/api -n ticketing

# Or manually scale old replica set
kubectl get rs -n ticketing
kubectl scale rs/api-xxxxx --replicas=3 -n ticketing

# Fix and retry deployment
```

## Monitoring and Maintenance

### Setup Prometheus Monitoring

```yaml
# Example ServiceMonitor for Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ticketing-metrics
  namespace: ticketing
spec:
  selector:
    matchLabels:
      app: api
  endpoints:
  - port: metrics
    interval: 30s
```

### Regular Tasks

- **Daily:** Check pod status, review logs, verify data consistency
- **Weekly:** Review and analyze metrics, check storage usage
- **Monthly:** Test backup/restore procedures, review security policies
- **Quarterly:** Update base images, audit dependencies, review architecture

## Related Documentation

- [README.md](../README.md) - Local development
- [docs/security/IMAGE_SCAN_REPORT.md](../docs/security/IMAGE_SCAN_REPORT.md) - Security scanning results
- [.github/workflows/](../.github/workflows/) - CI/CD pipeline
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [OpenShift Docs](https://docs.openshift.com/)
