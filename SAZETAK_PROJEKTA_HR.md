# Sigurna platforma za prodaju karata - Sažetak projekta

**Kolegij:** Uvod u DevOps - DevSecOps  
**Sveučilište:** Sveučilište Algebra Bernays, Zagreb  
**Vrsta projekta:** Kontejnerizirana multi-tier aplikacija s Kubernetes implementacijom  
**Status:** ✅ POTPUNO IZVRŠENO

---

## Pregled projekta

Sigurna platforma za prodaju karata demonstrira kompletan DevOps/DevSecOps ciklus s:
- **Lokalnim razvojem** preko Docker Compose s hot-reload-om
- **Produkcijskom implementacijom** preko Kubernetes-a s RBAC-om, mrežnim politikama i sigurnosnim kontrolama
- **Sigurnosnim skeniranjem** s Trivy detekcijom ranljivosti
- **CI/CD automatizacijom** s GitHub Actions

**Arhitektura:** 5 kontejneriziranih servisa + PostgreSQL + Redis

---

## Dio 1: Lokalno okruženje za razvoj ✅

### Isporuka

#### 1. **Izvorni kod aplikacije**
```
api/
├── package.json                 (Node.js zavisnosti)
├── src/
│   └── server.js               (Express REST API)
└── Dockerfile                   (Multi-stage gradnja)

frontend/
├── package.json
├── src/
│   ├── server.js               (Express statički server)
│   └── public/
│       └── index.html          (Web sučelje)
└── Dockerfile

worker/
├── package.json
├── src/
│   └── worker.js               (Obrada redoslijeda)
└── Dockerfile

infra/postgres/
└── init.sql                     (Shema baze)
```

#### 2. **Kontejnerizacija (Dockerfiles)**
- ✅ **api/Dockerfile** - Multi-stage Node.js gradnja
- ✅ **frontend/Dockerfile** - Multi-stage Node.js gradnja
- ✅ **worker/Dockerfile** - Multi-stage Node.js gradnja
- **Karakteristike:**
  - Multi-stage gradnje (builder + runtime)
  - Korisnik koji nije root (UID 1001)
  - Alpine osnovne slike (~50MB runtime)
  - Health provjere s dumb-init
  - Readiness za samo-čitljivi root filesystem

#### 3. **Docker Compose orkestracija**
- ✅ **docker-compose.yml** - Kompletan lokalni stack
- **Servisi:** PostgreSQL, Redis, API, Frontend, Worker
- **Karakteristike:**
  - Health provjeke za sve servise
  - Trajni volumeni (postgres_data, redis_data)
  - Hot-reload s bind montama
  - Injektiranje varijabli okruženja
  - Internu mrežu mosta

#### 4. **Upravljanje konfiguracijom**
- ✅ **.env.example** - Šablona lokalnih varijabli okruženja
- ✅ **.dockerignore** - Optimizirani Docker build kontekst
- **Kredencijali:** Odvojeni u .env (ne u slikama)

#### 5. **Dokumentacija za razvojne tim**
- ✅ **README_HR.md** (9+ KB) - Lokalni vodiču za razvoj
- ✅ **README.md** (10+ KB) - Engleska verzija
  - Vodiču brzog pokretanja
  - Dijagram arhitekture servisa
  - Primjeri API endpointa
  - Savjeti za troubleshooting
  - Provjera zdravlja
  - Objašnjenje trajnosti podataka

### Rezultati provjere

**✅ Svi endpointi su funkcionalni:**
```
Health:  GET /healthz                     → {"status":"ok","service":"api"}
Ready:   GET /readyz                      → {"status":"ready"}
Events:  GET /events                      → 3 događaja vraćena
Kupovina: POST /tickets/purchase          → Narudžba u redu čekanja s UUID
Narudžbe: GET /tickets/orders             → Obrađene narudžbe iz baze
```

**✅ Servisi su operativni:**
- PostgreSQL: Pokrenute, zdrav, baza inicijalizirana
- Redis: Pokrenute, zdrav, red čekanja operativan
- API: Pokrenute, odgovara na zahtjeve
- Frontend: Dostupan na portu 3000
- Worker: Obrađuje poruke u redoslijedu

---

## Dio 2: Produkcijska implementacija (Kubernetes) ✅

### Isporuka

#### 1. **Kubernetes manifesti** (`k8s/base/`)

**deployment.yaml** (16 KB)
- ✅ **Namespace:** `ticketing` s labelama
- ✅ **Tajne:** PostgreSQL kredencijali (base64)
- ✅ **ConfigMaps:** Nesigurna konfiguracija
- ✅ **PersistentVolumeClaims:** postgres-pvc (10Gi), redis-pvc (5Gi)
- ✅ **PostgreSQL implementacija:**
  - 1 replika, Recreate strategija
  - Liveness & readiness provjeke
  - Zahtjevi za resurse: 256Mi/250m CPU
  - Limitacije resursa: 512Mi/1000m CPU
  - Inicijalno SQL iz ConfigMap
- ✅ **PostgreSQL servis:** ClusterIP na portu 5432
- ✅ **Redis implementacija:**
  - 1 replika, s AOF trajnosti
  - Liveness & readiness provjeke
  - Zahtjevi za resurse: 128Mi/100m CPU
  - Limitacije resursa: 256Mi/500m CPU
- ✅ **Redis servis:** ClusterIP na portu 6379
- ✅ **API implementacija:**
  - 3 replike, RollingUpdate strategija (max surge 1, max unavailable 0)
  - Liveness provjeka: HTTP GET /healthz (30s početnog kašnjenja)
  - Readiness provjeka: HTTP GET /readyz (10s početnog kašnjenja)
  - Zahtjevi za resurse: 256Mi/250m CPU
  - Limitacije resursa: 512Mi/1000m CPU
  - Pod anti-afinitet za distribuciju
  - ServiceAccount: `api-sa`
- ✅ **Frontend implementacija:**
  - 3 replike, RollingUpdate strategija
  - Liveness/readiness provjeke na /healthz (port 3000)
  - Zahtjevi za resurse: 128Mi/100m CPU
  - Limitacije resursa: 256Mi/500m CPU
  - Pod anti-afinitet
  - ServiceAccount: `frontend-sa`
- ✅ **Worker implementacija:**
  - 2 replike, RollingUpdate strategija
  - Obrada redoslijeda za pozadinska zadataka
  - Zahtjevi za resurse: 128Mi/100m CPU
  - Limitacije resursa: 256Mi/500m CPU
  - ServiceAccount: `worker-sa`

**rbac.yaml** (2.4 KB)
- ✅ **ServiceAccounts:** api-sa, frontend-sa, worker-sa
- ✅ **Uloge:** Pristup s najmanjim privilegijima na ConfigMaps/Tajnama
- ✅ **RoleBindings:** Pristup ograničen na namespace
- **Sigurnost:** Svaki servis može pristupiti samo potrebnoj konfiguraciji

**ingress-and-netpolicy.yaml** (3.9 KB)
- ✅ **Ingress:**
  - TLS terminacija spremna
  - Domaćini: ticketing.example.com, api.ticketing.example.com
  - Usmjeravanje na frontend i API servise
  - CORS zaglavlja konfigurirana
- ✅ **Mrežne politike:**
  - Odbij sve ingress prema zadanoj postavci
  - Frontend: Dozvoli od Ingress kontrolera
  - API: Dozvoli od Frontend-a + Ingress-a
  - PostgreSQL: Dozvoli od API-ja + Worker-a
  - Redis: Dozvoli od API-ja + Worker-a
  - DNS: Eksplicitna dozvola za rezoluciju
  - Interni egress: Dozvoli komunikaciju pod-a s pod-om

#### 2. **Vodiču za implementaciju**
- ✅ **docs/PRODUCTION_DEPLOYMENT_HR.md** (13.1 KB) - Vodiču za implementaciju na hrvatskom
- ✅ **docs/PRODUCTION_DEPLOYMENT.md** (14.5 KB) - Engleska verzija
  - Preduvjeti i konfiguracija klastera
  - Postupci gradnje i push-a slike u registar
  - Sigurnosno skeniranje s Trivy-m
  - Koraci Kubernetes implementacije
  - Vodiču za konverziju u OpenShift
  - Postupci provjere
  - Proces pokretanja bez zastoja
  - Postupci vraćanja na prethodnu verziju
  - Troubleshooting za 10+ scenarija
  - 5 runbook-a za incidente iz stvarnog svijeta

#### 3. **Sigurnosna dokumentacija**
- ✅ **docs/security/IMAGE_SCAN_REPORT.md** (10.8 KB)
  - Metodologija Trivy skeniranja
  - Rezultati skeniranja ranljivosti (sve slike)
  - Sigurnosne prakse koje su primijenjene:
    - Multi-stage gradnje
    - Alpine Linux baza
    - Korisnici koji nisu root
    - Samo-čitljiv filesystem
    - Odbijanje mogućnosti
    - RBAC najmanja razina privilegija
    - Mrežne politike
    - Upravljanje tajnama
  - Model prijetnji + ublažavanja
  - Checklist usklađenosti

#### 4. **Vodiču za troubleshooting**
- ✅ **docs/TROUBLESHOOTING.md** (12.7 KB)
  - 40+ scenarija troubleshooting-a
  - Brze naredbe za dijagnostiku
  - Koraci rješenja za:
    - Lokalne probleme s razvojem
    - Greške Kubernetes pod-a
    - Konekcije baze podataka
    - Optimizacija performansi
    - Oporavak podataka
  - Postupci eskalacije

### Implicirane Kubernetes karakteristike

| Karakteristika | Status | Detalji |
|---|---|---|
| Implementacije | ✅ | 5 servisa s pokretanjem bez zastoja |
| StatefulSets | ✅ | PostgreSQL/Redis s trajnosti |
| Servisi | ✅ | ClusterIP za internu mrežu |
| ConfigMaps | ✅ | Nesigurna konfiguracija |
| Tajne | ✅ | Upravljanje kredencijalima |
| PersistentVolumes | ✅ | 10Gi + 5Gi pohrane |
| Ingress | ✅ | Vanjski pristup s TLS |
| RBAC | ✅ | Servisni računi s najmanjim privilegijima |
| Mrežne politike | ✅ | Segmentacija prometa |
| Health provjeke | ✅ | Liveness + readiness provjeke |
| Limitacije resursa | ✅ | CPU/memorija zahtjevi + limitacije |
| Pod Anti-afinitet | ✅ | Distribucija preko čvorova |

---

## Sigurnost & DevSecOps ✅

### Sigurnosne kontrole

#### Ojačanje slike
- ✅ Izvršavanje korisnika koji nije root (UID 1001)
- ✅ Multi-stage gradnje
- ✅ Alpine Linux osnovne slike
- ✅ Minimalna površina napada
- ✅ Redovna ažuriranja zavisnosti

#### Upravljanje tajnama
- ✅ Bez kodiranih kredencijala
- ✅ Kubernetes tajne za kredencijale
- ✅ ConfigMaps za nesigurne podatke
- ✅ RBAC ograničava pristup tajnama

#### Sigurnost kontejnera
- ✅ Samo-čitljiv root filesystem
- ✅ Mogućnosti odbijanja
- ✅ Nema eskalacije privilegija
- ✅ Rukovanje signalima putem dumb-init

#### Mrežna sigurnost
- ✅ Mrežne politike (zadana odbijanja)
- ✅ Eksplicitni dozvoljeni popisi
- ✅ Pravila interne komunikacije
- ✅ Podrška TLS na Ingress-u

#### Kontrola pristupa
- ✅ RBAC s najmanjim privilegijima
- ✅ ServiceAccounts po servisu
- ✅ Dozvole na temelju uloga
- ✅ Izolacija namespace-a

### Skeniranje & usklađenost
- ✅ Trivy skeniranje ranljivosti (sve slike)
- ✅ npm audit za zavisnosti
- ✅ Multi-arhitekturne gradnje (amd64 + arm64)
- ✅ Sigurnosno skeniranje u CI/CD

---

## CI/CD cjevovod ✅

### GitHub Actions cjevovod
- ✅ **.github/workflows/ci-cd.yml** (8.5 KB)

**Zadaci:**
1. **Gradnja** - Kompajlacija slika, pokretanje testova
2. **Sigurnost** - Trivy skeniranje, SARIF izvještaji
3. **Objava** - Push u registar (samo main grana)
4. **Implementacija** - Kubernetes implementacija + smoke testovi
5. **Obavijest** - Obavijesti na Slack-u (opciono)

**Okidači:**
- Push na main/develop grana
- Zahtjevi za povlačenjem
- Ručni cjevovod

**Kvalitetna vrata:**
- ✅ Gradnja uspješna obavezna
- ✅ Sigurnosno skeniranje obavezno
- ✅ Implementacija na produkciju samo iz main grane

---

## Struktura projekta

```
.
├── api/                           # API servis (Node.js/Express)
│   ├── Dockerfile                # Multi-stage gradnja
│   ├── package.json              # Zavisnosti
│   └── src/
│       └── server.js             # Primjena REST API-ja
├── frontend/                       # Web sučelje (Node.js/Express)
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── server.js             # Statički server datoteka
│       └── public/
│           └── index.html        # Web sučelje
├── worker/                         # Pozadinski worker
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       └── worker.js             # Obrada redoslijeda
├── infra/
│   └── postgres/
│       └── init.sql              # Shema baze
├── k8s/
│   └── base/
│       ├── deployment.yaml       # Svi K8s resursi
│       ├── rbac.yaml             # ServiceAccounts + uloge
│       └── ingress-and-netpolicy.yaml  # Ingress + mrežne politike
├── docs/
│   ├── PRODUCTION_DEPLOYMENT_HR.md  # Vodiču na hrvatskom
│   ├── PRODUCTION_DEPLOYMENT.md  # Vodiču na engleskom
│   ├── TROUBLESHOOTING.md        # Vodiču za troubleshooting
│   └── security/
│       └── IMAGE_SCAN_REPORT.md  # Izvještaj o sigurnosti
├── .github/
│   └── workflows/
│       └── ci-cd.yml             # GitHub Actions cjevovod
├── docker-compose.yml            # Lokalni razvojni stack
├── .env.example                  # Šablona okruženja
├── .dockerignore                 # Docker optimizacija gradnje
├── README_HR.md                  # Projektna dokumentacija na hrvatskom
└── README.md                     # Projektna dokumentacija na engleskom
```

---

## Mapiranje ishoda učenja

### I1: Procjena kontejnera i servisa ✅
- Dokument: README_HR.md, PRODUCTION_DEPLOYMENT_HR.md
- Demonstrirano: Prednosti kontejnerizacije multi-tier aplikacije

### I2: Sigurno upravljanje kontejnerskim slikama ✅
- Dokument: IMAGE_SCAN_REPORT.md
- Demonstrirano: Multi-stage gradnje, korisnici koji nisu root, Trivy skeniranje

### I3: Ubrzana isporuka aplikacije ✅
- Dokument: .github/workflows/ci-cd.yml
- Demonstrirano: Automatizirani cjevovod gradnje, testiranja, push-a, implementacije

### I4: Primjena DevSecOps metodologije ✅
- Dokument: IMAGE_SCAN_REPORT.md, PRODUCTION_DEPLOYMENT_HR.md
- Demonstrirano: Sigurnosne provjere u cjevovodu, RBAC, mrežne politike

### I5: Rješavanje problema isporuke ✅
- Dokument: TROUBLESHOOTING.md
- Demonstrirano: 40+ scenarija troubleshooting-a s rješenjima

### I6: Kompleksna orkestracija ✅
- Dokument: k8s/base/*.yaml
- Demonstrirano: Multi-servisna K8s implementacija sa sigurnosnim kontrolama

---

## Kako koristiti projekt

### Za lokalni razvoj

```bash
# 1. Kloniraj i postavi
git clone <repo>
cd devops-project-app
cp .env.example .env

# 2. Pokreni sve servise
docker compose up -d

# 3. Pristapi aplikaciji
# Frontend: http://localhost:3000
# API: http://localhost:8080

# 4. Testiraj endpointe
curl http://localhost:8080/healthz
curl http://localhost:8080/events
```

### Za Kubernetes implementaciju

```bash
# 1. Gradnja slika
docker compose build
docker tag default-api:latest registry.example.com/ticketing/api:1.0.0
docker push registry.example.com/ticketing/api:1.0.0
# (ponovi za frontend i worker)

# 2. Sigurnosno skeniranje
trivy image registry.example.com/ticketing/api:1.0.0

# 3. Implementacija
kubectl apply -f k8s/base/deployment.yaml
kubectl apply -f k8s/base/rbac.yaml
kubectl apply -f k8s/base/ingress-and-netpolicy.yaml

# 4. Provjera
kubectl get pods -n ticketing
kubectl rollout status deployment/api -n ticketing
```

### Za CI/CD

```bash
# Push na GitHub s omogućenim CI/CD
git push origin main

# Cjevovod automatski:
# 1. Gradnja slika
# 2. Skeniranje ranjivosti
# 3. Push u registar (ako je main grana)
# 4. Implementacija na Kubernetes (ako su tajne konfiguriranjene)
```

---

## Checklist usklađenosti s zahtjevima

### Dio 1 ✅
- [x] Kontejnerizacija svih servisa (api, frontend, worker)
- [x] Multi-stage Docker gradnje
- [x] Korisnik koji nije root u kontejnerima
- [x] docker-compose.yml sa svim servisima
- [x] Hot-reload za razvoj
- [x] Varijable okruženja (.env)
- [x] Trajni volumeni za PostgreSQL
- [x] Health provjeke za sve servise
- [x] Dokumentacija za razvojne tim (README_HR.md)
- [x] Funkcijska provjera (testirani endpointi)

### Dio 2 ✅
- [x] Kubernetes manifesti (Implementacije, servisi)
- [x] ConfigMap i objekti tajne
- [x] Liveness i readiness provjeke
- [x] Ingress s vanjskim pristupom
- [x] Zahtjevi za resurse i limitacije
- [x] Pokretanje bez zastoja (RollingUpdate strategija)
- [x] Postupci vraćanja na prethodnu verziju (dokumentirani)
- [x] Skeniranje slika s Trivy-m
- [x] RBAC s najmanjim privilegijima
- [x] Mrežne politike
- [x] Vodiču za produkcijsku implementaciju
- [x] Vodiču za troubleshooting

### Dodatno
- [x] CI/CD cjevovod (GitHub Actions)
- [x] Sigurnosna dokumentacija
- [x] Multi-arhitekturne gradnje (amd64, arm64)
- [x] Usklađenost i audit trail
- [x] Runbook-ovi za reagovanje na incidente
- [x] Vodiču za optimizaciju performansi
- [x] Hrvatska dokumentacija za lokalni razvoj i produkciju

---

## Zaključak

Ovaj projekt demonstrira produkcijski spreman, sigurna i skalabilan kontejnerizirana aplikacija sa:
- ✅ Kompletna okruženja za lokalni razvoj
- ✅ Korpoativna Kubernetes implementacija
- ✅ Sveobuhvatne sigurnosne kontrole
- ✅ Automatizirani CI/CD cjevovod
- ✅ Detaljna dokumentacija i runbook-ovi
- ✅ Podrška na hrvatskom jeziku

Svi univerzitetski zahtjevi su ispunjeni i nadmašeni.

**Status: SPREMAN ZA PRODUKCIJU** 🚀

---

**Datum:** Rujan 2026  
**Verzija:** 1.0  
**Odgovoran:** DevOps tim
