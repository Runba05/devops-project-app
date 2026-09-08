# Runbook za troubleshooting — Secure Event Ticketing Platform

Ovaj dokument opisuje stvarne incidente koji su se dogodili tijekom lokalnog razvoja (Docker Compose) i produkcijskog deploya (Kubernetes) ove aplikacije, uključujući dijagnostiku, uzrok i konačno rješenje svakog problema. Svi incidenti su reproducirani i riješeni tijekom stvarnog rada na projektu.

---

## Incident 1 — API kontejner "unhealthy" unatoč tome što aplikacija radi (Docker Compose)

**Simptom:**
`docker compose up -d` javlja:
```
dependency failed to start: container ticketing-api is unhealthy
```
iako logovi kontejnera pokazuju `API listening on port 8080`, a ručni poziv na `http://localhost:8080/healthz` s hosta vraća `200 OK`.

**Dijagnostika:**
1. `docker inspect ticketing-api --format='{{json .State.Health}}'` pokazuje ponavljajuću grešku:
   ```
   "Output":"Health check exceeded timeout (3s)"
   ```
2. Ručno pokretanje iste Node skripte unutar kontejnera (`docker exec`) radi trenutno i bez problema.
3. Zaključak: problem nije u aplikaciji, nego u tome kako Docker interno izvršava HEALTHCHECK naredbu.

**Uzrok:**
HEALTHCHECK naredba u Dockerfileu koristila je `http://localhost:8080/healthz`. Node.js unutar Alpine kontejnera ponekad prvo pokuša razriješiti `localhost` kao IPv6 adresu (`::1`). Ako server sluša samo na IPv4, spajanje na `::1` ne biva odmah odbijeno nego "visi" dok ne istekne definirani timeout (3s) — što izgleda kao da health check nikad ne uspijeva.

**Rješenje:**
Health check izmijenjen da koristi eksplicitnu IPv4 adresu `127.0.0.1` umjesto `localhost`, uz dodavanje eksplicitnog `process.exit()` i `.on('error', ...)` handlera:

```js
require('http').get('http://127.0.0.1:8080/healthz', (r) => {
  process.exit(r.statusCode === 200 ? 0 : 1);
}).on('error', () => process.exit(1));
```

Izmjena je napravljena i u `api/Dockerfile` (HEALTHCHECK direktiva) i u `docker-compose.yml` (`healthcheck.test` za servis `api`), jer je compose definicija imala prioritet nad onom iz Dockerfilea.

**Validacija:** `docker compose ps` prikazuje `ticketing-api` kao `healthy`.

---

## Incident 2 — Frontend ne može dohvatiti listu eventova ("Failed to fetch") (Docker Compose)

**Simptom:**
Stranica na `http://localhost:3000` prikazuje prazan dropdown za odabir eventa i grešku:
```json
{ "error": "Failed to initialize page", "details": "Failed to fetch" }
```

**Dijagnostika:**
1. Chrome DevTools → Console prikazuje:
   ```
   api:8080/events:1  Failed to load resource: net::ERR_NAME_NOT_RESOLVED
   ```
2. Frontend server ima endpoint `/config` koji vraća `apiBaseUrl` iz env varijable `API_BASE_URL`.
3. Ta varijabla je u `docker-compose.yml` postavljena na `http://api:8080` — ispravno za komunikaciju **između kontejnera** unutar Docker mreže, ali klijentski JavaScript kod izvršava se u **browseru na hostu**, koji hostname `api` ne može razriješiti.

**Uzrok:**
Miješanje dvije razine mreže: interna Docker Compose mreža (gdje `api` postoji kao DNS ime) i host mreža (gdje browser radi i gdje `api` ne postoji).

**Rješenje:**
Promijenjena vrijednost `API_BASE_URL` za `frontend` servis u `docker-compose.yml` na `http://localhost:8080`, budući da API port (`8080`) izlazi na host preko `ports:` mapiranja.

**Validacija:** Dropdown na `http://localhost:3000` popunjen eventima; kupnja karte vraća `{"message": "Order queued", "orderId": "..."}`.

---

## Incident 3 — Kubernetes pods u `ErrImagePull` / `ImagePullBackOff`

**Simptom:**
Nakon `kubectl apply -f k8s/base/deployment.yaml`, pods za `api`, `frontend` i `worker` ostaju u statusu `ErrImagePull` / `ImagePullBackOff`.

**Dijagnostika:**
```
kubectl get deployment api -n ticketing -o jsonpath="{.spec.template.spec.containers[0].image}"
→ ticketing-api:latest

kubectl get deployment api -n ticketing -o jsonpath="{.spec.template.spec.containers[0].imagePullPolicy}"
→ Always
```
Lokalno izgrađene slike (preko `docker compose build`) imale su naziv `devopsproject-api:latest` (Docker Compose imenuje slike po nazivu direktorija projekta), dok su K8s manifesti očekivali `ticketing-api:latest`. Uz to, `imagePullPolicy: Always` je tjerao Kubernetes da uvijek pokuša povući sliku s udaljenog registryja, čak i kad bi lokalna slika s odgovarajućim imenom postojala.

**Uzrok:**
Nepodudaranje naziva slika između build procesa (Compose) i deployment manifesta (K8s), u kombinaciji s pull policyjem koji ignorira lokalni cache.

**Rješenje:**
1. Označene postojeće lokalne slike novim imenima koje manifest očekuje:
   ```
   docker tag devopsproject-api:latest ticketing-api:latest
   docker tag devopsproject-frontend:latest ticketing-frontend:latest
   docker tag devopsproject-worker:latest ticketing-worker:latest
   ```
2. Promijenjen `imagePullPolicy` u `deployment.yaml` s `Always` na `IfNotPresent` za sva tri servisa.
3. Primijenjen ažurirani manifest i restartani deploymenti.

**Validacija:** `kubectl get pods -n ticketing` prikazuje sve pods u statusu `Running`.

**Napomena za produkciju:** U pravom produkcijskom okruženju slike bi trebale biti objavljene u pravi container registry (npr. GitHub Container Registry, Docker Hub, ili privatni registry), s konzistentnim imenovanjem i verzioniranim tagovima — `imagePullPolicy: IfNotPresent` na lokalnom Docker Desktop klasteru je prihvatljivo za razvojno/testno okruženje, ali ne zamjenjuje pravi image pipeline.

---

## Incident 4 — PostgreSQL pod u `CrashLoopBackOff` s greškom "Operation not permitted"

**Simptom:**
Pod `postgres-*` u Kubernetesu neprestano pada. Logovi pokazuju:
```
chmod: /var/lib/postgresql/data: Operation not permitted
chmod: /var/run/postgresql: Operation not permitted
initdb: error: could not change permissions of directory "/var/lib/postgresql/data": Operation not permitted
```

**Dijagnostika:**
1. Deployment ima ispravno definiran `securityContext` (`runAsUser: 999`, `fsGroup: 999`), što bi na standardnom Kubernetesu trebalo osigurati ispravno vlasništvo volumena.
2. `kubectl get pvc -n ticketing` i `kubectl get pv` potvrđuju da se koristi `standard` StorageClass, koji na Docker Desktop Kubernetesu koristi `hostpath` provisioner.
3. Poznato ograničenje: `hostpath` volumeni na Docker Desktopu (posebno na Windows hostu) ne poštuju uvijek `fsGroup` postavku ispravno, pa proces koji radi kao non-root korisnik (999) nema dozvolu mijenjati vlasništvo/dozvole direktorija koji je montiran s pogrešnim vlasnikom (obično root).

**Uzrok:**
Ograničenje `hostpath` storage provisionera na Docker Desktop Kubernetesu — `fsGroup` iz `securityContext` ne primjenjuje se pouzdano na takve volumene.

**Rješenje:**
Dodan `initContainer` u `postgres` deployment koji radi **kao root** (`runAsUser: 0`) prije glavnog kontejnera, i ručno postavlja ispravno vlasništvo i dozvole na direktorij prije nego glavni PostgreSQL proces (koji radi kao user 999) pokuša pisati u njega:

```yaml
initContainers:
- name: fix-permissions
  image: busybox:1.36
  command: ["sh", "-c", "chown -R 999:999 /var/lib/postgresql/data && chmod -R 700 /var/lib/postgresql/data"]
  volumeMounts:
  - mountPath: /var/lib/postgresql/data
    name: postgres-storage
    subPath: postgres
  securityContext:
    runAsUser: 0
```

**Validacija:** `kubectl get pods -n ticketing` prikazuje `postgres` pod u statusu `1/1 Running` bez restartova.

**Napomena za produkciju:** Na pravom produkcijskom Kubernetes/OpenShift klasteru (s odgovarajućim CSI storage driverom, npr. AWS EBS, Azure Disk, ili OpenShift-ov default storage), `fsGroup` obično radi ispravno bez potrebe za ovim workaroundom. Ovaj initContainer je zadržan kao dodatna zaštita koja ne šteti ni u okruženjima gdje `fsGroup` radi ispravno.

---

## Incident 5 — API pod ne može razriješiti Redis hostname (`getaddrinfo EAI_AGAIN redis`)

**Simptom:**
Nakon što su postgres i imagePull problemi riješeni, `api` pod i dalje pada u `CrashLoopBackOff`. Logovi pokazuju:
```
Redis error: Connection timeout
Redis error: getaddrinfo EAI_AGAIN redis
```

**Dijagnostika:**
1. `kubectl get svc -n ticketing` potvrđuje da servis `redis` postoji s ispravnim imenom, ClusterIP-om i portom.
2. `kubectl get networkpolicy -n ticketing` prikazuje sve očekivane politike, uključujući `allow-dns-egress`.
3. `kubectl describe networkpolicy allow-dns-egress -n ticketing` pokazuje da politika dopušta DNS promet (port 53/UDP) samo prema namespaceu koji ima label `name=kube-system`.
4. `kubectl get namespace kube-system --show-labels` pokazuje da `kube-system` namespace ima label `kubernetes.io/metadata.name=kube-system`, ali **ne** i `name=kube-system`.

**Uzrok:**
NetworkPolicy je pisan s pretpostavkom da svaki namespace ima label `name=<namespace>`, što je uobičajena konvencija na nekim distribucijama Kubernetesa, ali nije zajamčeno na svima. Na Docker Desktop Kubernetesu `kube-system` po defaultu nema taj label, samo standardni `kubernetes.io/metadata.name`. Posljedično, `allow-dns-egress` politika nikad nije odgovarala `kube-system` namespaceu, pa je sav DNS promet (uključujući razrješavanje internih servisnih imena poput `redis`) bio blokiran za sve pods u `ticketing` namespaceu koji podliježu default-deny politici.

**Rješenje:**
Ručno dodan label koji politika očekuje:
```
kubectl label namespace kube-system name=kube-system
```
Zatim restartani pogođeni deploymenti (`api`) da ponovno pokušaju DNS lookup.

**Validacija:** `kubectl logs <api-pod>` više ne prikazuje Redis greške; `kubectl get pods -n ticketing` prikazuje sve api pods kao `1/1 Running`.

**Napomena za produkciju:** Ovo je važan nalaz za bilo koji klaster koji koristi NetworkPolicy temeljen na `namespaceSelector` s labelom `name=...` — potrebno je unaprijed provjeriti (ili eksplicitno postaviti) da target namespace doista ima taj label, umjesto pretpostavljati da ga svaka K8s distribucija automatski dodjeljuje.

---

## Incident 6 — Frontend u Kubernetesu ne može dohvatiti evente (identičan simptom kao Incident 2, drugi uzrok)

**Simptom:**
Nakon uspješnog port-forwardanja (`kubectl port-forward svc/frontend 3000:3000` i `svc/api 8080:8080`), stranica na `http://localhost:3000` ponovno prikazuje:
```json
{ "error": "Failed to initialize page", "details": "Failed to fetch" }
```
s istom greškom u konzoli: `api:8080/events — ERR_NAME_NOT_RESOLVED`.

**Dijagnostika:**
1. `kubectl get deployment frontend -n ticketing -o jsonpath="{.spec.template.spec.containers[0].env}"` pokazuje da `API_BASE_URL` dolazi iz ConfigMapa `ticketing-config`.
2. `kubectl get configmap ticketing-config -n ticketing -o jsonpath="{.data.API_BASE_URL}"` vraća `http://api:8080` — interni K8s DNS naziv servisa, koji (isto kao u Incidentu 2) browser na hostu ne može razriješiti kad se pristupa preko port-forwarda.

**Uzrok:**
Isti konceptualni problem kao Incident 2 (miješanje interne mrežne adrese s adresom dostupnom klijentu), ali ovaj put na razini Kubernetes ConfigMapa umjesto Docker Compose environment varijable.

**Rješenje:**
Ažurirana vrijednost u ConfigMapu:
```
kubectl patch configmap ticketing-config -n ticketing --type merge -p '{"data":{"API_BASE_URL":"http://localhost:8080"}}'
```
Zatim restartan `frontend` deployment da povuče novu vrijednost (ConfigMap izmjene ne primjenjuju se automatski na već pokrenute pods):
```
kubectl rollout restart deployment frontend -n ticketing
```

**Validacija:** Uz aktivan `kubectl port-forward svc/api 8080:8080 -n ticketing` u zasebnom terminalu, stranica na `http://localhost:3000` uspješno prikazuje listu eventova i omogućuje kupnju karte.

**Napomena za produkciju:** Ovaj problem ne bi postojao u pravom produkcijskom scenariju gdje frontend poslužuje statične datoteke, a JavaScript u browseru pristupa API-ju preko javno izloženog Ingress/Route hostname-a (npr. `https://api.ticketing.example.com`) umjesto internog servisnog imena. `API_BASE_URL` treba biti okolišno-specifičan: interni DNS naziv za server-to-server pozive, javni hostname za sve adrese koje se šalju klijentskom (browser) kodu.

---

## Opći zaključci i preporuke

1. **Uvijek razlikovati "internu" i "eksternu" mrežnu adresu.** Varijable poput `API_BASE_URL` koje se koriste u kodu koji se izvršava u browseru moraju sadržavati adresu dostupnu klijentu, ne internu Docker/K8s mrežnu adresu — čak i ako je ta varijabla ispravna za server-to-server komunikaciju.
2. **Health check naredbe unutar kontejnera trebaju koristiti eksplicitne IPv4 adrese** (`127.0.0.1`) umjesto `localhost`, radi izbjegavanja IPv6 rezolucijskih kašnjenja u minimalnim (Alpine) slikama.
3. **Nazivi Docker slika moraju biti usklađeni** između build alata (Docker Compose) i deployment manifesta (Kubernetes) — po mogućnosti kroz zajednički CI/CD pipeline koji gradi i tagira slike jednoznačno, umjesto ručnog imenovanja.
4. **NetworkPolicy pravila temeljena na namespace labelima zahtijevaju provjeru** da ciljani namespace doista ima očekivani label — ne pretpostavljati da svaka K8s distribucija automatski dodjeljuje label `name=<namespace>`.
5. **Permission problemi na perzistentnim volumenima** su čest izazov na lokalnim/hostpath-based storage rješenjima; `initContainer` koji eksplicitno postavlja vlasništvo prije starta glavne aplikacije je robusno i prenosivo rješenje koje radi bez obzira na ponašanje `fsGroup`-a na danom storage provисioneru.
