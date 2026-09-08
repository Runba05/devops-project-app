# Sigurna platforma za prodaju karata - Lokalni razvoj

> Projekt za kolegij **Uvod u DevOps - DevSecOps** na Sveučilištu Algebra Bernays, Zagreb

## Pregled projekta

Ovo je multi-tier aplikacija za upravljanje prodajom karata za događaje sa sljedećim servisima:

- **Frontend** - Web sučelje za pregled događaja i kupovinu karata (port 3000)
- **API** - REST API za događaje, narudžbe i health provjere (port 8080)
- **Worker** - Pozadinska obrada redoslijeda poruka
- **PostgreSQL** - Trajna pohrana narudžbi (port 5432)
- **Redis** - Red čekanja i cache (port 6379)

## Arhitektura

```
┌──────────────────────────────────────────────────────────┐
│                   Frontend (Node.js/Express)              │
│                     Port 3000 - Web sučelje                │
└─────────────────┬──────────────────────────────────────────┘
                   │ HTTP/REST
                   ▼
┌──────────────────────────────────────────────────────────┐
│                   API (Node.js/Express)                   │
│              Port 8080 - REST endpointi                    │
└──────┬──────────────────────────────┬──────────────────────┘
       │                              │
       ▼                              ▼
┌──────────────────┐        ┌──────────────────┐
│   PostgreSQL      │        │     Redis        │
│   Port 5432        │        │   Port 6379      │
│   Pohrana          │        │  Red + Cache     │
└──────────┬─────────┘        └─────────┬─────────┘
           │                            │
           └────────────┬───────────────┘
                         ▼
              ┌────────────────────────┐
              │  Worker (Node.js)      │
              │  Obrada redoslijeda    │
              └────────────────────────┘
```

## Brzi početak

### Preduvjeti

- Docker Desktop 4.20+ ili Docker Engine + Docker Compose 2.20+
- Git
- Internetska veza (za preuzimanje slika)

### Instalacija i pokretanje

1. **Kloniraj repozitorij:**
   ```bash
   git clone https://github.com/Runba05/devops-project-app.git
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

## Provjera funkcioniranja

### Health provjere

```bash
# API je dostupan
curl http://localhost:8080/healthz
```

Odgovor:
```json
{"status": "ok", "service": "api"}
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
5. Klikni **Purchase**
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

## Upravljanje servisima

### Pokretanje

```bash
# Pokreni s gradnjom (prvi put ili nakon promjena u Dockerfile)
docker compose up --build -d

# Pokreni bez gradnje (brže)
docker compose up -d
```

### Zaustavljanje

```bash
# Zaustavi sve servise (podaci ostaju)
docker compose stop

# Zaustavi i ukloni kontejnere (volumeni ostaju)
docker compose down

# Potpuno čišćenje (uklanja i volumene)
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

Projekt je konfiguriran za automatsko osvježavanje koda pri promjenama:

```yaml
volumes:
  - ./api/src:/app/src        # Sinkronizira promjene u kodu
  - /app/node_modules         # Sprječava prepletanje
```

Kada se naprave promjene u `api/src/`, `frontend/src/` ili `worker/src/`, aplikacija se automatski restarta zahvaljujući `nodemon`.

## Trajnost podataka

### Volumeni

- `postgres_data` - PostgreSQL baza (opstaje nakon `docker compose down`)
- `redis_data` - Redis snapshot (opstaje nakon `docker compose down`)

### Resetiranje podataka

```bash
# Potpuno čišćenje sa brisanjem volumena
docker compose down -v

# Ponovno pokretanje s novom bazom
docker compose up -d
```

## Troubleshooting

Za detaljnu dijagnostiku i rješenja stvarnih problema na koje se naišlo tijekom razvoja ovog projekta (health check timeout, mrežni problemi, konfiguracija API adrese), pogledati `docs/TROUBLESHOOTING.md`.

Osnovne dijagnostičke naredbe:

```bash
# Provjeri logove
docker compose logs

# Restart svih servisa
docker compose restart

# Potpuni restart
docker compose down -v
docker compose up --build -d

# Provjeri mrežu
docker network inspect ticketing-network
```

### Port već korišten

```bash
# Promijeni portove u .env datoteci
API_PORT=8081
FRONTEND_PORT=3001
POSTGRES_PORT=5433

# Ili pronađi i ugasi proces na portu (Windows)
netstat -ano | findstr :8080
taskkill /PID <PID> /F
```

## Sigurnosne karakteristike (lokalni razvoj)

- Korisnik koji nije root (UID 1001) u svim Node.js kontejnerima
- Multi-stage Docker gradnje (smanjena veličina slika)
- Health provjere za sve servise
- Tajne u `.env` datoteci (nisu hardkodirane u kodu)
- Mrežna izolacija preko Docker mosta

**Napomena:** Lokalni `.env` ima zadanu lozinku iz `.env.example` — nikada ne koristiti tu vrijednost u produkciji.

## Sljedeći koraci

- **Produkcija (Kubernetes):** vidi [`PRODUCTION_DEPLOYMENT_HR.md`](PRODUCTION_DEPLOYMENT_HR.md)
- **Rješavanje problema:** vidi [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md)
