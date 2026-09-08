# Vodič za produkcijsku implementaciju — Sigurna platforma za prodaju karata

Ovaj dokument opisuje kako je aplikacija Secure Event Ticketing Platform implementirana i testirana na Kubernetesu (Docker Desktop Kubernetes), te što bi bilo potrebno za prijelaz na pravi produkcijski klaster.

## Preduvjeti

- Kubernetes klaster dostupan preko `kubectl` (u ovom projektu: Docker Desktop Kubernetes, uključen preko Settings → Kubernetes → Enable Kubernetes)
- Docker slike izgrađene lokalno (`docker compose build`) i označene imenima koja koriste K8s manifesti (`ticketing-api`, `ticketing-frontend`, `ticketing-worker`)

Provjera pristupa klasteru:

```bash
kubectl version
kubectl get nodes
```

## Redoslijed primjene manifesta

Manifesti se nalaze u `k8s/base/` i primjenjuju se sljedećim redoslijedom (prvi fajl kreira `ticketing` namespace koji ostali koriste):

```bash
kubectl apply -f k8s/base/deployment.yaml
kubectl apply -f k8s/base/rbac.yaml
kubectl apply -f k8s/base/ingress-and-netpolicy.yaml
```

`deployment.yaml` sadrži: namespace, ConfigMap i Secret objekte, PersistentVolumeClaim-ove za PostgreSQL i Redis, te Deployment i Service resurse za svih pet servisa (postgres, redis, api, frontend, worker).

`rbac.yaml` sadrži: ServiceAccount, Role i RoleBinding resurse za api, frontend i worker servise, svaki s minimalnom potrebnom razinom pristupa.

`ingress-and-netpolicy.yaml` sadrži: Ingress resurs za vanjski pristup te sedam NetworkPolicy resursa (default-deny-ingress i eksplicitna dopuštenja za potrebnu komunikaciju između servisa).

## Priprema lokalnih Docker slika za Kubernetes

Budući da Docker Compose imenuje slike po nazivu direktorija projekta (npr. `devopsproject-api`), a K8s manifesti očekuju nazive `ticketing-*`, lokalno izgrađene slike potrebno je označiti odgovarajućim imenima prije primjene manifesta:

```bash
docker compose build
docker tag devopsproject-api:latest ticketing-api:latest
docker tag devopsproject-frontend:latest ticketing-frontend:latest
docker tag devopsproject-worker:latest ticketing-worker:latest
```

Manifesti koriste `imagePullPolicy: IfNotPresent`, čime Kubernetes koristi lokalno dostupnu sliku umjesto pokušaja povlačenja s udaljenog registryja.

## Provjera statusa nakon implementacije

```bash
kubectl get pods -n ticketing
kubectl get svc -n ticketing
```

Svih devet pod-ova (1× postgres, 1× redis, 3× api, 3× frontend, 2× worker — broj replika ovisi o trenutnoj konfiguraciji) trebalo bi biti u statusu `Running` s oznakom `1/1` ili `3/3` spremnih kontejnera.

## Pristup aplikaciji

Bez konfiguriranog Ingress kontrolera (koji Docker Desktop Kubernetes ne uključuje po defaultu), aplikaciji se pristupa preko `kubectl port-forward`, otvorenog u zasebnim terminalima za svaki servis:

```bash
kubectl port-forward svc/frontend 3000:3000 -n ticketing
kubectl port-forward svc/api 8080:8080 -n ticketing
```

Zatim je aplikacija dostupna na `http://localhost:3000`.

**Napomena o konfiguraciji API adrese:** ConfigMap `ticketing-config` sadrži varijablu `API_BASE_URL` koju frontend server prosljeđuje klijentskom (browser) kodu preko `/config` endpointa. Za pristup preko port-forwarda ta vrijednost mora biti `http://localhost:8080` (adresa dostupna browseru), a ne interno servisno ime `http://api:8080` koje vrijedi samo za komunikaciju između pod-ova unutar klastera. Ovo je detaljnije opisano u `docs/TROUBLESHOOTING.md` (Incident 6).

## Testiranje rolling update i rollback mehanizma

Primjer promjene konfiguracije bez prekida rada aplikacije:

```bash
kubectl rollout restart deployment/api -n ticketing
kubectl rollout status deployment/api -n ticketing
```

Provjera povijesti i vraćanje na prethodnu verziju:

```bash
kubectl rollout history deployment/api -n ticketing
kubectl rollout undo deployment/api -n ticketing
```

## Rješavanje problema

Za dijagnostiku i rješenja stvarnih problema na koje se naišlo tijekom implementacije (health check timeout, permission problemi na PostgreSQL volumenu, DNS blokada zbog NetworkPolicy-ja i konfiguracija API adrese), pogledati `docs/TROUBLESHOOTING.md`.

Osnovne dijagnostičke naredbe:

```bash
kubectl get pods -n ticketing
kubectl describe pod <ime-poda> -n ticketing
kubectl logs <ime-poda> -n ticketing
kubectl get events -n ticketing --sort-by='.lastTimestamp'
```

## Što bi trebalo dodati za pravi produkcijski klaster

Ovaj projekt je testiran na lokalnom Docker Desktop Kubernetesu, koji ima određena ograničenja u odnosu na pravi produkcijski klaster (npr. hostpath storage umjesto pravog CSI drivera, nema ugrađenog Ingress kontrolera niti TLS certifikata). Za stvaran produkcijski deployment bilo bi potrebno dodatno:

- Objaviti Docker slike u pravi container registry (npr. GitHub Container Registry) s verzioniranim tagovima umjesto `latest`, te automatizirati taj proces kroz CI/CD (`.github/workflows/ci-cd.yml`).
- Instalirati i konfigurirati Ingress kontroler (npr. nginx-ingress) s TLS certifikatima (npr. preko cert-managera).
- Koristiti StorageClass koji odgovara stvarnom cloud provideru (npr. AWS EBS, Azure Disk) umjesto lokalnog hostpath provisionera.
- Ako se koristi OpenShift umjesto čistog Kubernetesa, zamijeniti Ingress resurs s OpenShift Route objektom.
- Redovito (npr. u CI/CD pipelineu) pokretati Trivy skeniranje na svaku novu verziju slike prije objave.
