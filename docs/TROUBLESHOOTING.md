# Troubleshooting Runbook - Secure Event Ticketing Platform

Quick reference guide for common issues and solutions.

## Issues Quick Index

| Symptom | Cause | Solution |
|---------|-------|----------|
| Frontend won't load | API not responding | See: API Service Down |
| Orders not processing | Worker crashed or Redis full | See: Queue Issues |
| Database connection fails | PostgreSQL unavailable or wrong credentials | See: Database Issues |
| High memory usage | Resource leak or under-provisioned | See: Performance Issues |
| Pods stuck in Pending | Not enough resources | See: Cluster Issues |

---

## Local Development (docker-compose)

### Issue: Services fail to start

**Check logs:**
```bash
docker compose logs
docker compose logs api
docker compose logs postgres
```

**Solution:**
```bash
# Stop all and restart
docker compose down
docker compose up -d

# Or with rebuild
docker compose down -v
docker compose up --build
```

### Issue: Port already in use

**Problem:** Error like "Address already in use"

**Solution:**
```bash
# Change ports in .env
POSTGRES_PORT=5433
API_PORT=8081
FRONTEND_PORT=3001

# Or kill existing process
lsof -i :8080
kill -9 <PID>

# Then restart
docker compose restart
```

### Issue: Database won't initialize

**Check logs:**
```bash
docker compose logs postgres
```

**Solution:**
```bash
# Reset database
docker compose down -v
docker compose up

# Or manually reinit
docker compose exec postgres psql -U ticketing_user -d ticketing -f /docker-entrypoint-initdb.d/init.sql
```

### Issue: Can't connect to services from host

**Test connectivity:**
```bash
curl http://localhost:8080/healthz
telnet localhost 5432
redis-cli -p 6379 PING
```

**Solution:**
```bash
# Check docker network
docker network inspect ticketing-network

# Restart containers
docker compose restart

# Check port mappings
docker compose ps
```

---

## Kubernetes Production

### Issue: Pods not starting (CreateContainerConfigError)

**Diagnose:**
```bash
kubectl describe pod <pod-name> -n ticketing
kubectl logs <pod-name> -n ticketing
```

**Common causes and fixes:**

**1. Secret not found**
```bash
# Check if secret exists
kubectl get secrets -n ticketing

# If missing, create it
kubectl create secret generic ticketing-credentials \
  --from-literal=POSTGRES_USER=user \
  --from-literal=POSTGRES_PASSWORD=pass \
  -n ticketing
```

**2. ConfigMap not found**
```bash
# Check configmaps
kubectl get configmaps -n ticketing

# Apply if missing
kubectl apply -f k8s/base/deployment.yaml
```

**3. Image pull error**
```bash
# Check image exists
docker images | grep ticketing

# Or in registry
curl -s https://registry.example.com/v2/ticketing/api/tags/list

# Solution: Build and push image
docker compose build
docker tag default-api:latest registry.example.com/ticketing/api:1.0.0
docker push registry.example.com/ticketing/api:1.0.0

# Update deployment
kubectl set image deployment/api api=registry.example.com/ticketing/api:1.0.0 -n ticketing
```

### Issue: Pod stuck in CrashLoopBackOff

**Check logs:**
```bash
kubectl logs -f <pod-name> -n ticketing --tail=100
```

**Common causes:**

**1. Application error**
- Check logs for stack trace
- Fix code and redeploy
```bash
# Redeploy
kubectl rollout restart deployment/api -n ticketing
```

**2. Dependency unavailable**
```bash
# Check if postgres is ready
kubectl exec deployment/api -n ticketing -- sh -c 'nc -zv postgres 5432'

# Check if redis is ready
kubectl exec deployment/api -n ticketing -- sh -c 'nc -zv redis 6379'

# Restart dependencies first
kubectl rollout restart deployment/postgres -n ticketing
kubectl rollout restart deployment/redis -n ticketing

# Then restart app
kubectl rollout restart deployment/api -n ticketing
```

**3. Resource limit exceeded**
```bash
# Check memory usage
kubectl top pods -n ticketing

# Increase limits
kubectl set resources deployment/api \
  --limits=memory=1Gi,cpu=2 \
  --requests=memory=512Mi,cpu=1 \
  -n ticketing

# Restart
kubectl rollout restart deployment/api -n ticketing
```

### Issue: Persistent Volume stuck in Pending

**Check:**
```bash
kubectl get pvc -n ticketing
kubectl describe pvc postgres-pvc -n ticketing
```

**Solution:**

**1. Storage class missing**
```bash
# List storage classes
kubectl get storageclass

# If none, create default
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/generic-storage
EOF
```

**2. Node disk full**
```bash
# Check node disk usage
kubectl describe node

# Free space:
# - Delete old pods/images
# - Scale down applications temporarily
# - Add more nodes
```

**3. Storage provisioner unavailable**
```bash
# Check provisioner status
kubectl get deployment -n kube-system

# For EBS/EFS/GCE provisioner, ensure it's running
kubectl logs -n kube-system <provisioner-pod>
```

### Issue: API returns 503 Service Unavailable

**Diagnose readiness:**
```bash
kubectl exec deployment/api -n ticketing -- \
  wget -O- http://localhost:8080/readyz
```

**Common causes:**

**1. Database down**
```bash
# Check postgres
kubectl get pods -l app=postgres -n ticketing

# Check logs
kubectl logs deployment/postgres -n ticketing

# Restart
kubectl rollout restart deployment/postgres -n ticketing
```

**2. Redis down**
```bash
# Check redis
kubectl exec deployment/redis -n ticketing -- redis-cli PING

# Restart if needed
kubectl rollout restart deployment/redis -n ticketing
```

**3. Network connectivity**
```bash
# From API pod, test both
kubectl exec deployment/api -n ticketing -- sh -c 'nc -zv postgres 5432'
kubectl exec deployment/api -n ticketing -- sh -c 'nc -zv redis 6379'

# If fails, check network policies
kubectl get networkpolicies -n ticketing
```

### Issue: Orders stuck in "queued" status

**Check worker status:**
```bash
# Worker pods running?
kubectl get pods -l app=worker -n ticketing

# Worker logs
kubectl logs -l app=worker -n ticketing --tail=50
```

**Check Redis queue:**
```bash
# Queue length
kubectl exec deployment/redis -n ticketing -- \
  redis-cli LLEN ticket_orders

# Peek at queue
kubectl exec deployment/redis -n ticketing -- \
  redis-cli LPOP ticket_orders
```

**Solutions:**

**1. Scale up workers**
```bash
kubectl scale deployment worker --replicas=5 -n ticketing

# Monitor
watch kubectl exec deployment/redis -n ticketing -- redis-cli LLEN ticket_orders

# Scale back
kubectl scale deployment worker --replicas=2 -n ticketing
```

**2. Worker crashing**
```bash
kubectl describe pod -l app=worker -n ticketing
kubectl logs -l app=worker -n ticketing
```

**3. Database connection issue**
```bash
# Check if table exists
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "\\dt"

# Check permissions
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "SELECT * FROM ticket_orders LIMIT 1;"
```

### Issue: Out of memory (OOMKilled)

**Check resource usage:**
```bash
kubectl top pods -n ticketing

# Check limits
kubectl get pods -o json -n ticketing | \
  jq '.items[] | {name: .metadata.name, limits: .spec.containers[].resources.limits}'
```

**Solution:**

**1. Increase limits**
```bash
kubectl set resources deployment/api \
  --limits=memory=1Gi,cpu=2000m \
  --requests=memory=512Mi,cpu=1000m \
  -n ticketing
```

**2. Optimize code**
- Check for memory leaks
- Profile application
- Reduce dataset size

**3. Scale horizontally**
```bash
kubectl scale deployment api --replicas=5 -n ticketing
```

### Issue: High latency / slow requests

**Profile:**
```bash
# Time a request
time curl http://localhost:8080/events

# Check metrics
kubectl top pods -n ticketing
kubectl top nodes

# Check network
kubectl describe networkpolicies -n ticketing
```

**Solutions:**

**1. Database slow queries**
```bash
# Enable slow query log
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c \
  "ALTER SYSTEM SET log_min_duration_statement = 1000;"

# Restart postgres
kubectl rollout restart deployment/postgres -n ticketing

# Check logs
kubectl logs deployment/postgres -n ticketing | grep duration
```

**2. Add caching**
- Implement Redis caching in API
- Cache frequently accessed data

**3. Scale up**
```bash
kubectl scale deployment api --replicas=5 -n ticketing
```

### Issue: Frontend can't reach API

**Test connectivity from frontend pod:**
```bash
kubectl exec deployment/frontend -n ticketing -- \
  curl http://api:8080/events

# If fails, check DNS
kubectl exec deployment/frontend -n ticketing -- \
  nslookup api

# Check network policy
kubectl get networkpolicies -n ticketing
```

**Solution:**

**1. Update API_BASE_URL**
```bash
kubectl patch configmap ticketing-config -n ticketing \
  -p '{"data":{"API_BASE_URL":"http://api:8080"}}'

# Restart frontend
kubectl rollout restart deployment/frontend -n ticketing
```

**2. Fix network policy**
```bash
# Check if ingress to API is allowed
kubectl describe networkpolicies -n ticketing

# Apply fix
kubectl apply -f k8s/base/ingress-and-netpolicy.yaml

# Restart pods
kubectl rollout restart deployment/frontend -n ticketing
```

### Issue: Rolling update fails

**Check rollout status:**
```bash
kubectl rollout status deployment/api -n ticketing

# Check pod events
kubectl describe pod -l app=api -n ticketing
```

**Solution: Rollback**
```bash
# Immediate rollback
kubectl rollout undo deployment/api -n ticketing

# Monitor
kubectl rollout status deployment/api -n ticketing

# Check if working
curl http://localhost:8080/healthz
```

**Solution: Manual intervention**
```bash
# Scale old replica set
kubectl get rs -n ticketing
kubectl scale rs/api-xxxxx --replicas=3 -n ticketing

# Scale new one to 0
kubectl scale rs/api-yyyyy --replicas=0 -n ticketing
```

---

## Data Recovery

### Backup Database

```bash
# Quick backup
kubectl exec deployment/postgres -n ticketing -- \
  pg_dump -U ticketing_user -d ticketing > backup.sql

# Scheduled backup (CronJob)
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: ticketing
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:16-alpine
            command:
            - sh
            - -c
            - pg_dump -U \$POSTGRES_USER -d \$POSTGRES_DB | gzip > /backups/backup-\$(date +%s).sql.gz
            env:
            - name: PGHOST
              value: postgres
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: ticketing-credentials
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ticketing-credentials
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              value: ticketing
            volumeMounts:
            - name: backup
              mountPath: /backups
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
EOF
```

### Restore Database

```bash
# From backup file
kubectl exec -i deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing < backup.sql

# Verify
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "SELECT COUNT(*) FROM ticket_orders;"
```

---

## Performance Tuning

### Database Optimization

```bash
# Analyze tables
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "ANALYZE;"

# Check index usage
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c \
  "SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;"

# Reindex if needed
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "REINDEX DATABASE ticketing;"
```

### Redis Optimization

```bash
# Check memory usage
kubectl exec deployment/redis -n ticketing -- \
  redis-cli INFO memory

# Clear expired keys
kubectl exec deployment/redis -n ticketing -- \
  redis-cli FLUSHDB

# Persistence check
kubectl exec deployment/redis -n ticketing -- \
  redis-cli LASTSAVE
```

---

## Escalation Path

1. **Developer:** Check logs, restart pods, basic debugging
2. **DevOps Engineer:** Investigate infrastructure, scaling, networking
3. **DBA:** Database performance, backup/restore, schema optimization
4. **Security Team:** For security incidents or breach investigations
5. **Incident Commander:** For P1 incidents affecting production

**Contact:** On-call rotation via PagerDuty
