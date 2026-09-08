# Vodič za produkcijsku implementaciju - Sigurna platforma za prodaju karata

Ovaj vodič pokriva implementaciju aplikacije na Kubernetes/OpenShift u produkciji.

## Sadržaj

1. [Preduvjeti prije implementacije](#preduvjeti-prije-implementacije)
2. [Gradnja slika i push u registar](#gradnja-slika-i-push-u-registar)
3. [Sigurnosno skeniranje](#sigurnosno-skeniranje)
4. [Kubernetes implementacija](#kubernetes-implementacija)
5. [OpenShift implementacija](#openshift-implementacija)
6. [Provjera rada](#provjera-rada)
7. [Pokretanje bez zastoja](#pokretanje-bez-zastoja)
8. [Vraćanje na prethodnu verziju](#vraćanje-na-prethodnu-verziju)
9. [Troubleshooting](#troubleshooting)
10. [Runbook za incidente](#runbook-za-incidente)

## Preduvjeti prije implementacije

### Okruženje

- Kubernetes 1.24+ ili OpenShift 4.12+
- `kubectl` ili `oc` CLI instaliran
- Docker CLI za gradnju slika
- Pristup registru slike (Docker Hub, ECR, GCR, Quay, itd.)
- Helm 3.x+ (opciono)

### Konfiguracija klastera

```bash
# Provjeri pristup klasteru
kubectl cluster-info
kubectl get nodes

# Kreiraj namespace (ili koristi postojeći)
kubectl create namespace ticketing
```

### Potrebne tajne

Prije implementacije, pripremi kredencijale:

```bash
# Kreiraj tajnu za Docker registar (ako je privatna registra)
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=korisnik \
  --docker-password=lozinka \
  --docker-email=email@example.com \
  -n ticketing

# Provjeri tajnu
kubectl get secrets -n ticketing
```

### Kontroler Ingress-a

Provjeri je li kontroler ingress-a instaliran:

```bash
# Za nginx ingress kontroler
kubectl get deployment -n ingress-nginx

# Ako nije instalirano, instaliraj ga:
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

## Gradnja slika i push u registar

### Korak 1: Gradnja Docker slika

```bash
# Gradnja svih slika lokalno
docker compose build

# Označavanje slika za registar
docker tag default-api:latest registry.example.com/ticketing/api:1.0.0
docker tag default-frontend:latest registry.example.com/ticketing/frontend:1.0.0
docker tag default-worker:latest registry.example.com/ticketing/worker:1.0.0

# Ili koristi buildx za multi-arhitekturu (arm64, amd64)
docker buildx build --platform linux/amd64,linux/arm64 -t registry.example.com/ticketing/api:1.0.0 ./api --push
docker buildx build --platform linux/amd64,linux/arm64 -t registry.example.com/ticketing/frontend:1.0.0 ./frontend --push
docker buildx build --platform linux/amd64,linux/arm64 -t registry.example.com/ticketing/worker:1.0.0 ./worker --push
```

### Korak 2: Push u registar

```bash
# Prijava u registar
docker login registry.example.com

# Push slika
docker push registry.example.com/ticketing/api:1.0.0
docker push registry.example.com/ticketing/frontend:1.0.0
docker push registry.example.com/ticketing/worker:1.0.0

# Provjera push-a
docker images registry.example.com/ticketing/*
```

## Sigurnosno skeniranje

### Korištenje Trivy

```bash
# Instalacija Trivy
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
apt-get update
apt-get install trivy

# Skeniranje slika lokalno
trivy image localhost:5000/ticketing/api:1.0.0
trivy image localhost:5000/ticketing/frontend:1.0.0
trivy image localhost:5000/ticketing/worker:1.0.0

# Generiranje izvještaja
trivy image --format json --output api-scan.json registry.example.com/ticketing/api:1.0.0

# Izlaz sa greškom ako postoje kritične ranljivosti
trivy image --exit-code 1 --severity HIGH,CRITICAL registry.example.com/ticketing/api:1.0.0
```

### Saniranje

Ako su pronađene ranljivosti:

1. **Ažuriranje osnovnih slika:**
   ```bash
   # U Dockerfile
   FROM node:20-alpine  # Ažuriraj na najnoviju verziju
   ```

2. **Ažuriranje zavisnosti:**
   ```bash
   npm audit
   npm audit fix
   npm audit fix --force  # Koristi oprezno
   ```

3. **Ponovno graditi i skenirati:**
   ```bash
   docker compose build --no-cache
   trivy image localhost:5000/ticketing/api:latest
   ```

## Kubernetes implementacija

### Korak 1: Pripremi manifeste

Ažuriraj reference slika u `k8s/base/deployment.yaml`:

```yaml
# Promijeni sa 'latest' na specifičnu verziju
image: registry.example.com/ticketing/api:1.0.0
image: registry.example.com/ticketing/frontend:1.0.0
image: registry.example.com/ticketing/worker:1.0.0

# Ako koristiš privatnu registru, dodaj imagePullSecrets
imagePullSecrets:
- name: regcred
```

### Korak 2: Konfiguriraj kredencijale

Uredi `k8s/base/deployment.yaml` i ažuriraj Secret:

```bash
# Kreiraj nove kredencijale
kubectl create secret generic ticketing-credentials \
  --from-literal=POSTGRES_USER=prod_user \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -base64 32)" \
  -n ticketing --dry-run=client -o yaml > /tmp/secret.yaml

# Provjeri i primijeni
cat /tmp/secret.yaml
kubectl apply -f /tmp/secret.yaml
```

### Korak 3: Implementacija na Kubernetes

```bash
# Primijeni manifeste redom
kubectl apply -f k8s/base/deployment.yaml
kubectl apply -f k8s/base/rbac.yaml
kubectl apply -f k8s/base/ingress-and-netpolicy.yaml

# Provjera implementacije
kubectl get all -n ticketing
kubectl get pvc -n ticketing
```

### Korak 4: Čekaj da se servisi pokrenu

```bash
# Nadgledi pokretanje
kubectl rollout status deployment/postgres -n ticketing
kubectl rollout status deployment/redis -n ticketing
kubectl rollout status deployment/api -n ticketing
kubectl rollout status deployment/frontend -n ticketing
kubectl rollout status deployment/worker -n ticketing
```

## OpenShift implementacija

### Korak 1: Kreiraj OpenShift projekt

```bash
oc new-project ticketing

# Ili koristi postojeći projekt
oc project ticketing
```

### Korak 2: Konverzija Ingress u Route

Zamijeni Ingress u `k8s/base/ingress-and-netpolicy.yaml` sa OpenShift Route:

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

### Korak 3: Implementacija

```bash
# Primijeni manifeste
oc apply -f k8s/base/deployment.yaml
oc apply -f k8s/base/rbac.yaml
oc apply -f k8s/base/openshift-routes.yaml

# Nadgledaj
oc get all
oc describe route frontend-route
```

## Provjera rada

### Provjera statusa pod-a

```bash
# Svi pod-i trebaju biti Running i Ready
kubectl get pods -n ticketing

# Provjera detalja pod-a
kubectl describe pod -l app=api -n ticketing
kubectl logs -l app=api -n ticketing --tail=50
```

### Testiranje API endpointa

```bash
# Dohvati API pod IP
API_POD=$(kubectl get pod -l app=api -o jsonpath='{.items[0].metadata.name}' -n ticketing)

# Proslijedi port
kubectl port-forward pod/$API_POD 8080:8080 -n ticketing &

# Testiraj endpointe
curl http://localhost:8080/healthz
curl http://localhost:8080/events
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"test@example.com","quantity":1}'
curl http://localhost:8080/tickets/orders

# Zaustavi prosljeđivanje porta
pkill -f "port-forward"
```

### Provjera trajnih volumena

```bash
# Список PVC
kubectl get pvc -n ticketing

# Provjera PostgreSQL podataka
kubectl exec -it deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "SELECT COUNT(*) FROM ticket_orders;"
```

## Pokretanje bez zastoja

### Ažuriranje slike

```bash
# Uredi deployment za korištenje nove slike
kubectl set image deployment/api api=registry.example.com/ticketing/api:2.0.0 -n ticketing

# Nadgledi pokretanje
kubectl rollout status deployment/api -n ticketing

# Provjeri povijest
kubectl rollout history deployment/api -n ticketing
```

### Ažuriranje varijabli okruženja

```bash
# Ažuriraj ConfigMap
kubectl patch configmap ticketing-config -n ticketing \
  -p '{"data":{"API_BASE_URL":"http://api-v2:8080"}}'

# Restartaj pod-e da preuzmu novije postavke
kubectl rollout restart deployment/api -n ticketing
kubectl rollout status deployment/api -n ticketing
```

## Vraćanje na prethodnu verziju

### Vraćanje implementacije

```bash
# Provjeri povijest
kubectl rollout history deployment/api -n ticketing

# Vrati se na prethodnu verziju
kubectl rollout undo deployment/api -n ticketing

# Vrati se na specifičnu verziju
kubectl rollout undo deployment/api --to-revision=2 -n ticketing

# Provjera vraćanja
kubectl rollout status deployment/api -n ticketing
```

### Provjera integriteta podataka nakon vraćanja

```bash
# Povezivanja na PostgreSQL
kubectl exec -it deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing

# Query narudžbi
SELECT * FROM ticket_orders ORDER BY created_at DESC LIMIT 10;
```

## Troubleshooting

### Pod-ovi zapeti u Pending

```bash
# Provjera resursa čvora
kubectl top nodes
kubectl describe nodes

# Provjera PVC vezivanja
kubectl describe pvc postgres-pvc -n ticketing

# Provjera događaja
kubectl get events -n ticketing --sort-by='.lastTimestamp'
```

### Pod-ovi u Crash Loop

```bash
# Provjeri logove
kubectl logs -f deployment/api -n ticketing

# Opiši pod
kubectl describe pod -l app=api -n ticketing

# Provjera limitacija resursa
kubectl get resourcequota -n ticketing
```

### Greške pri konekciji na bazu

```bash
# Test konekcije iz API pod-a
kubectl exec deployment/api -n ticketing -- \
  sh -c 'nc -zv postgres 5432'

# Provjera DNS rezolucije
kubectl exec deployment/api -n ticketing -- \
  nslookup postgres

# Provjera kredencijala
kubectl get secret ticketing-credentials -n ticketing -o yaml
```

## Runbook za incidente

### Scenarij 1: API servis nije dostupan

**Simptomi:** Frontend ne može dosegnuti API, zahtjevi isteknu

**Dijagnostika:**
```bash
# 1. Provjera statusa pod-a
kubectl get pods -l app=api -n ticketing

# 2. Provjera logova
kubectl logs -l app=api -n ticketing --tail=100

# 3. Provjera servisa
kubectl get svc api -n ticketing

# 4. Test konekcije
kubectl exec deployment/frontend -n ticketing -- curl http://api:8080/healthz
```

**Rješenje:**
```bash
# Restart API implementacije
kubectl rollout restart deployment/api -n ticketing

# Ili vraćanje ako je nedavno ažurirano
kubectl rollout undo deployment/api -n ticketing

# Nadgledaj
kubectl rollout status deployment/api -n ticketing
```

### Scenarij 2: Baza podataka puna / nema mjesta

**Simptomi:** Narudžbe se ne obrađuju, greške baze

**Dijagnostika:**
```bash
# Provjera veličine baze
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "SELECT pg_size_pretty(pg_database_size('ticketing'));"

# Provjera dostupnog mjesta
kubectl exec deployment/postgres -n ticketing -- df -h /var/lib/postgresql/data
```

**Rješenje:**
```bash
# Proširi PVC
kubectl patch pvc postgres-pvc -n ticketing \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Ili obriši stare podatke
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "DELETE FROM ticket_orders WHERE created_at < NOW() - INTERVAL '90 days';"
```

### Scenarij 3: Visoka memorijska potrošnja

**Simptomi:** Pod-ovi su OOMKilled (izlazni kod 137)

**Dijagnostika:**
```bash
# Provjera potrošnje resursa
kubectl top pods -n ticketing

# Provjera limitacija
kubectl get pods -o json -n ticketing | jq '.items[].spec.containers[].resources'
```

**Rješenje:**
```bash
# Povećaj limitacije memorije
kubectl set resources deployment/api \
  --limits=memory=1Gi,cpu=2 \
  -n ticketing

# Restart pod-a
kubectl rollout restart deployment/api -n ticketing
```

### Scenarij 4: Red čekanja se nakuplja (narudžbe se ne obrađuju)

**Simptomi:** Redis red čekanja raste, narudžbe su zapete u "queued" stanju

**Dijagnostika:**
```bash
# Provjera veličine Redis reda čekanja
kubectl exec deployment/redis -n ticketing -- \
  redis-cli LLEN ticket_orders

# Provjera logova worker-a
kubectl logs -l app=worker -n ticketing --tail=50

# Provjera baze za obrađene narudžbe
kubectl exec deployment/postgres -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "SELECT COUNT(*) FROM ticket_orders WHERE status='processed';"
```

**Rješenje:**
```bash
# Preskaliranje worker replika
kubectl scale deployment worker --replicas=5 -n ticketing

# Nadgledaj obradu
watch kubectl exec deployment/redis -n ticketing -- redis-cli LLEN ticket_orders

# Preskaliranje natrag nakon što je red čekanja obrađen
kubectl scale deployment worker --replicas=2 -n ticketing
```

---

**Status:** ✅ Spreman za produkcijsku implementaciju
