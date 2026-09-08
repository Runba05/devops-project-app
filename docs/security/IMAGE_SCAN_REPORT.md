# Sigurnosno izvješće – skeniranje kontejnerskih slika

## Sažetak

Ovaj dokument prikazuje rezultate stvarnog skeniranja ranjivosti alatom Trivy nad tri kontejnerske slike izgrađene za ovaj projekt (`ticketing-api`, `ticketing-frontend`, `ticketing-worker`), te sigurnosne prakse primijenjene u Dockerfileovima i Kubernetes manifestima.

**Datum skeniranja:** 8. rujna 2026.
**Alat:** Trivy (aquasec/trivy, preuzeto i pokrenuto lokalno preko Dockera)
**Skenirane slike:** `ticketing-api:latest`, `ticketing-frontend:latest`, `ticketing-worker:latest` (sve na bazi `node:20-alpine`)

## Naredba korištena za skeniranje

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ticketing-api:latest
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ticketing-frontend:latest
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ticketing-worker:latest
```

## Rezultati skeniranja

### Sloj operacijskog sustava (Alpine 3.23.4)

Sve tri slike dijele identičan rezultat na razini OS-a, budući da su izgrađene na istoj baznoj slici (`node:20-alpine`):

| Ozbiljnost | Broj |
|---|---|
| CRITICAL | 0 |
| HIGH | 4 |
| MEDIUM | 14 |
| LOW | 32 |
| **Ukupno** | **50** |

Sve HIGH i MEDIUM ranjivosti odnose se na pakete `libcrypto3` i `libssl3` (OpenSSL), primjerice CVE-2026-14456 (Denial of Service preko neograničenog rasta memorije u QUIC serveru) i CVE-2026-45447 (Heap Use-After-Free u PKCS7_verify()). Za sve postoje popravljene verzije (3.5.7-r0 / 3.5.8-r0).

### Sloj Node.js paketa (node-pkg)

| Slika | CRITICAL | HIGH | MEDIUM | LOW | Ukupno |
|---|---|---|---|---|---|
| ticketing-api | 1 | 19 | 9 | 3 | 32 |
| ticketing-frontend | 1 | 19 | 8 | 3 | 31 |
| ticketing-worker | 1 | 19 | 6 | 3 | 29 |

**Važna napomena o izvoru ovih nalaza:** Velika većina ovih ranjivosti (uključujući jedinu CRITICAL — CVE-2026-59873 u paketu `tar`) nalazi se u putanji `usr/local/lib/node_modules/npm/node_modules/...` — to je **npm CLI alat ugrađen u baznu `node:20-alpine` sliku**, koji se koristi isključivo tijekom `npm install` u build fazi (builder stage), a ne u runtime okruženju aplikacije. Kontejneri se pokreću s `ENTRYPOINT ["dumb-init", "--"]` i `CMD ["npm", "start"]`, gdje `npm start` samo pokreće `node src/server.js` — sam npm CLI alat se nakon toga ne izvršava, pa te ranjivosti ne predstavljaju stvarnu izloženost u produkciji.

**Ranjivosti u stvarnim runtime ovisnostima aplikacije** (paketi pod `app/node_modules/`, koje aplikacija stvarno učitava i koristi):

| Paket | CVE | Ozbiljnost | Instalirana verzija | Popravljena verzija |
|---|---|---|---|---|
| qs (ovisnost Express-a) | CVE-2026-82417, CVE-2026-82562 | MEDIUM | 6.15.3 | 6.16.0 |
| uuid | CVE-2026-41907 | HIGH | 10.0.0 | 11.1.1 / 12.0.1 / 13.0.1 |

Ostale runtime ovisnosti koje aplikacija stvarno koristi (`express`, `redis`, `pg`, `dotenv`, i njihove tranzitivne ovisnosti izvan `qs`/`uuid`) nemaju prijavljenih ranjivosti (`0` u stupcu Vulnerabilities za odgovarajuće `package.json` datoteke).

## Sigurnosne prakse stvarno primijenjene u projektu

Sljedeće prakse su provjerene u stvarnom sadržaju Dockerfileova i Kubernetes manifesta korištenih u ovom projektu:

- **Multi-stage build** — svaki servis (api, frontend, worker) koristi dvije faze: `builder` (instalacija ovisnosti) i runtime faza koja kopira samo `node_modules` i izvorni kod, bez alata za razvoj.
- **Non-root korisnik** — svi Node.js kontejneri pokreću se pod korisnikom `nodejs` (UID 1001), kreiranim eksplicitno u Dockerfileu (`addgroup`/`adduser`), a PostgreSQL pod korisnikom 999.
- **dumb-init za rukovanje signalima** — svi Node.js kontejneri koriste `dumb-init` kao PID 1 proces radi ispravnog rukovanja SIGTERM signalima pri gašenju.
- **RBAC s najmanjom razinom privilegija** — svaki servis (api, frontend, worker) ima zaseban ServiceAccount, Role i RoleBinding u `k8s/base/rbac.yaml`, primijenjen i testiran na Kubernetesu.
- **Segmentacija mreže** — `k8s/base/ingress-and-netpolicy.yaml` sadrži `default-deny-ingress` politiku te eksplicitna dopuštenja samo za potrebnu komunikaciju između servisa; ove politike su primijenjene i njihovo ispravno funkcioniranje potvrđeno je tijekom rješavanja stvarnog DNS incidenta opisanog u `docs/TROUBLESHOOTING.md`.
- **Bez hardkodiranih tajni** — kredencijali za PostgreSQL prosljeđuju se kroz Kubernetes Secret objekte i `.env` datoteku u lokalnom razvoju (temeljem `.env.example` predloška), nikada izravno u izvornom kodu.

## Preporučene mjere saniranja

Na temelju stvarnih rezultata skeniranja, sljedeće mjere bi popravile pronađene ranjivosti:

1. **Ažurirati baznu sliku** na noviju `node:20-alpine` verziju (ili prijeći na `node:22-alpine`) kako bi se povukle novije verzije `libssl3`/`libcrypto3` paketa (3.5.8-r0 ili noviji) koje popravljaju sve pronađene OpenSSL CVE-ove.
2. **Ažurirati `uuid` paket** na verziju 11.1.1 ili noviju u `package.json` datotekama servisa koji ga koriste (`npm install uuid@latest`).
3. **Ažurirati `qs` paket** (tranzitivna ovisnost Express-a) na 6.16.0 ili noviju verziju, ili ažurirati sam Express na noviju verziju koja povlači ispravljenu verziju `qs`.
4. **Ranjivosti u ugrađenom npm CLI alatu** (unutar bazne slike) ne zahtijevaju hitnu akciju budući da se npm ne izvršava u runtime okruženju kontejnera, no problem će se prirodno riješiti ažuriranjem na noviju baznu Node.js sliku koja dolazi s novijom verzijom npm-a.

## Napomena o opsegu ovog izvješća

Ovo skeniranje je izvršeno ručno, jednokratno, tijekom razvoja projekta. Nije uspostavljen automatizirani (CI/CD) proces redovitog skeniranja pri svakoj novoj verziji slike — to je naveden kao preporučeni sljedeći korak u `PRODUCTION_DEPLOYMENT_HR.md`.
