# Sigurna platforma za prodaju karata - Lokalni razvoj

> Uzorni projekt za kolegij **Uvod u DevOps - DevSecOps** na Sveučilištu Algebra Bernays, Zagreb

## Pregled projekta

Ovo je multi-tier aplikacija za upravljanje prodajom karata za događaje sa sljedećim servisima:

- **Frontend** - Web sučelje za pregled događaja i kupovinu karata (port 3000)
- **API** - REST API za događaje, narudžbe i health provjere (port 8080)
- **Worker** - Pozadinska obrada redoslijeda poruka
- **PostgreSQL** - Trajna pohrana narudžbi (port 5432)
- **Redis** - Red čekanja i cache (port 6379)

## Arhitektura

```
┌─────────────────────────────────────────────────────────────┐
│                   Frontend (Node.js/Express)                │
│                     Port 3000 - Web sučelje                  │
└────────────────┬────────────────────────────────────────────┘
                 │ HTTP/REST
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                   API (Node.js/Express)                     │
│              Port 8080 - REST endpointi                      │
└─────┬──────────────────────────────┬────────────────────────┘
      │                              │
      ▼                              ▼
┌──────────────────┐        ┌──────────────────┐
│   PostgreSQL     │        │     Redis        │
│   Port 5432      │        │   Port 6379      │
│   Pohrana        │        │  Red + Cache     │
└──────────────────┘        └──────────────────┘
      ▲                              ▲
      │                              │
      └──────────────┬───────────────┘
                     │
                     ▼
        ┌──────────────────────────┐
        │  Worker (Node.js)        │
        │  Obrada redoslijeda      │
        └──────────────────────────┘
```

## Brzi početak

### Preduvjeti

- Docker Desktop 4.20+ ili Docker Engine + Docker Compose 2.20+
- Git
- Internetska veza (za preuzimanje slika)

### Instalacija i pokretanje

1. **Kloniraj repozitorij:**
   ```bash
   git clone https://github.com/matej-basic/devops-project-app.git
   cd devops-project-app
   ```

2. **Kreiraj .env datoteku:**
   ```bash
   cp .env.example .env
   ```

3. **Pokreni sve servise:**
   ```bash
   docker compose up -d
   ```

Aplikacija je dostupna na:
- **Frontend UI:** http://localhost:3000
- **API:** http://localhost:8080

##验证funkcioniranja

### Health provjere

```bash
# API je dostupan
curl http://localhost:8080/healthz
```

Odgovor:
```json
{"status": "ok", "service": "api"}
```

```bash
# API je spreman (sve zavisnosti dostupne)
curl http://localhost:8080/readyz
```

Odgovor:
```json
{"status": "ready"}
```

### Pregled događaja

```bash
curl http://localhost:8080/events
```

Odgovor:
```json
[
  {
    "id": "evt-1001",
    "name": "DevSecOps Bootcamp",
    "location": "Zagreb",
    "availableTickets": 150
  },
  {
    "id": "evt-1002",
    "name": "Cloud Native Day",
    "location": "Split",
    "availableTickets": 200
  },
  {
    "id": "evt-1003",
    "name": "Security Engineering Meetup",
    "location": "Rijeka",
    "availableTickets": 90
  }
]
```

### Kupovina karata

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{
    "eventId": "evt-1001",
    "customerEmail": "student@example.com",
    "quantity": 2
  }'
```

Odgovor:
```json
{
  "message": "Order queued",
  "orderId": "96e7eb49-2815-4642-bda9-f5ad09bfd047"
}
```

### Pregled obrađenih narudžbi

```bash
curl http://localhost:8080/tickets/orders
```

Odgovor:
```json
[
  {
    "order_id": "96e7eb49-2815-4642-bda9-f5ad09bfd047",
    "event_id": "evt-1001",
    "customer_email": "student@example.com",
    "quantity": 2,
    "status": "processed",
    "created_at": "2026-03-05T19:58:40.954Z"
  }
]
```

### Web sučelje

1. Otvori http://localhost:3000
2. Odaberi događaj iz padajućeg izbornika
3. Unesi email adresu
4. Unesi broj karata
5. Klikni **Kupite**
6. Vidi potvrdu s ID-om narudžbe

## Konfiguracija okruženja

### Lokalne varijable (.env datoteka)

```env
# Baza podataka
POSTGRES_DB=ticketing
POSTGRES_USER=ticketing_user
POSTGRES_PASSWORD=change_me_local
POSTGRES_HOST=postgres
POSTGRES_PORT=5432

# Cache/Red
REDIS_HOST=redis
REDIS_PORT=6379

# Servisi
API_PORT=8080
FRONTEND_PORT=3000
API_BASE_URL=http://api:8080

# Red čekanja
QUEUE_NAME=ticket_orders

# Okruženje
NODE_ENV=development
```

### Promjena konfiguracije

```bash
# Promijeni port ako je već zauzet
POSTGRES_PASSWORD=moja_lozinka docker compose up
```

## Upravljanje servisima

### Pokretanje

```bash
# Pokreni s gradnjom (prvi put ili nakon promjena u Dockerfile)
docker compose up --build

# Pokreni bez gradnje (brže)
docker compose up

# Pokreni u pozadini
docker compose up -d
```

### Zaustavljanje

```bash
# Zaustavi sve servise (podaci ostaju)
docker compose stop

# Zaustavi i ukloni kontejnere (volumeni ostaju)
docker compose down

# Potpuna čišćenja (uklanja i volumene)
docker compose down -v
```

### Pregled statusa

```bash
# Prikaži sve servise
docker compose ps

# Vidi logove
docker compose logs

# Vidi logove određenog servisa
docker compose logs api
docker compose logs postgres
```

## Hot-reload za razvoj

Projekat je konfiguriran za automatsko osvježavanje kod promjena:

```yaml
volumes:
  - ./api/src:/app/src        # Sinkronizira promjene u kodu
  - /app/node_modules         # Sprječava prepletanje
```

Kada napraviš promjene u `api/src/`, `frontend/src/` ili `worker/src/`, aplikacija će se automatski restartati zahvaljujući `nodemon`.

## Trajnost podataka

### Volumeni

- `postgres_data` - PostgreSQL baza (opstaje nakon `docker compose down`)
- `redis_data` - Redis snapshot (opstaje nakon `docker compose down`)

### Resetiranje podataka

```bash
# Potpuno čišćenje sa brisanjem volumena
docker compose down -v

# Ponovno pokretanje s novom bazom
docker compose up
```

## Troubleshooting

### Servisi se ne pokreću

```bash
# Provjeri logove
docker compose logs

# Restart svih servisa
docker compose restart

# Potpuni restart
docker compose down -v
docker compose up --build
```

### Port već korišten

```bash
# Promijeni portove u .env datoteci
API_PORT=8081
FRONTEND_PORT=3001
POSTGRES_PORT=5433

# Ili ubij proces na portu
# Linux/macOS
lsof -i :8080
kill -9 <PID>

# Windows
netstat -ano | findstr :8080
taskkill /PID <PID> /F
```

### Baza se ne inicijalizira

```bash
# Vidi PostgreSQL logove
docker compose logs postgres

# Resetiraj i pokušaj ponovno
docker compose down -v
docker compose up
```

### Redis veza ne radi

```bash
# Restart Redis
docker compose restart redis

# Provjeri logove
docker compose logs redis
```

### Kontejneri se često ne poklapaju s lokalnom mrežom

```bash
# Provjeri mrežu
docker network inspect ticketing-network

# Restart svih servisa
docker compose restart

# Provjeri portove
docker compose ps
```

## Sigurnosne karakteristike (lokalni razvoj)

- ✅ Korisnik koji nije root (UID 1001) u svim kontejnerima
- ✅ Multi-stage Docker gradnje (smanjeni redak slika)
- ✅ Health provjere za sve servise
- ✅ Tajne u `.env` datoteci (ne u kodu)
- ✅ Mrežna izolacija preko Docker mostu
- ⚠️ Napomena: Lokalni `.env` ima zadanu lozinku - nikada ne koristiti u produkciji!

## Savjeti za performanse

1. **Koristiti nazvane volumene za bazu:**
   - Brže od bind mountova na Docker Desktop za macOS/Windows
   - Već konfigurirano u `docker-compose.yml`

2. **Onemogućiti hot-reload ako nije potreban:**
   ```bash
   docker compose up --no-build
   ```

3. **Nadgledati potrošnju resursa:**
   ```bash
   docker stats
   ```

## Sljedeći koraci

- **Produkcija:** Vidi [`docs/PRODUCTION_DEPLOYMENT_HR.md`](docs/PRODUCTION_DEPLOYMENT_HR.md)
- **Sigurnost:** Vidi [`docs/security/IMAGE_SCAN_REPORT.md`](docs/security/IMAGE_SCAN_REPORT.md)
- **CI/CD:** Vidi [`.github/workflows/`](.github/workflows/)

## Kontakt i podrška

Za pitanja o lokalnom razvoju:
1. Provjeri logove: `docker compose logs`
2. Vidi troubleshooting sekciju gore
3. Kontaktiraj na email institucije

---

**Status:** ✅ Spreman za lokalni razvoj
