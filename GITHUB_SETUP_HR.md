# GitHub Setup - Detaljne upute

## KORAK 1: Kreiraj GitHub račun (ako ga nemaš)

1. Idi na https://github.com/signup
2. Unesi:
   - Email adresu
   - Lozinku
   - Korisničko ime (npr: matej-basic)
3. Klikni "Create account"
4. Potvrdi email

---

## KORAK 2: Kreiraj novi repozitorij na GitHub-u

1. Logiraj se na https://github.com
2. Klikni "+" u gornjem desnom kutu
3. Odaberi "New repository"
4. Unesi podatke:
   - **Repository name:** `devops-project-app`
   - **Description:** "Sigurna platforma za prodaju karata - DevOps/DevSecOps projekt"
   - **Public** ← VAŽNO! Mora biti javna da profesor može pristupiti
   - Ostavi ostalo kao jest
5. Klikni "Create repository"

Sada trebaš link poput: https://github.com/TVOJE_KORISNICKO_IME/devops-project-app

---

## KORAK 3: Konfiguracija Git-a na komputeru

Prije nego što pushаš, konfigurira Git s tvojim GitHub kredencijalima:

```bash
# Postavi svoje ime
git config --global user.name "Tvoje Ime"

# Postavi email (koristi isti kao na GitHub-u)
git config --global user.email "tvoj.email@example.com"
```

---

## KORAK 4: Pushaj projekt na GitHub

Pokreni ove komande U TVOM PROJEKTU (devops-project-app direktoriju):

```bash
# 1. Inicijalizacija Git-a (ako već nije done)
git init

# 2. Dodaj sve datoteke
git add .

# 3. Kreiraj prvi commit
git commit -m "Inicijalni commit: Sigurna platforma za prodaju karata - DevOps/DevSecOps projekt"

# 4. Preimenuj granu na 'main' (ako trebam)
git branch -M main

# 5. Dodaj udaljeni repozitorij (zamijeni TVOJE_KORISNICKO_IME)
git remote add origin https://github.com/TVOJE_KORISNICKO_IME/devops-project-app.git

# 6. Pushaj na GitHub
git push -u origin main
```

---

## KORAK 5: Provjera da je sve učitano

1. Idi na https://github.com/TVOJE_KORISNICKO_IME/devops-project-app
2. Trebao bi vidjeti sve datoteke:
   - docker-compose.yml
   - k8s/base/ folder
   - docs/ folder
   - Dockerfiles
   - README.md i README_HR.md
   - itd.

---

## KORAK 6: Pošalji profesoru

Pošalji profesoru ovaj link:

```
https://github.com/TVOJE_KORISNICKO_IME/devops-project-app
```

Ili ako koristiš email, pošalji ovako:

```
Predmet: Predaja DevOps projekta

Cijenjeni profesore,

Predajem svoj projekt na:
https://github.com/TVOJE_KORISNICKO_IME/devops-project-app

Projekt sadrži:
- Dio 1: Lokalni razvoj (docker-compose)
- Dio 2: Kubernetes produkcija
- Sigurnosno skeniranje
- CI/CD automatizacija
- Dokumentacija na hrvatskom i engleskom

Gdje početi:
1. README_HR.md (lokalni setup)
2. SAZETAK_PROJEKTA_HR.md (pregled)
3. PRODUCTION_DEPLOYMENT_HR.md (produkcija)

Za testiranje:
docker compose up

Hvala,
[Tvoje ime]
```

---

## ČESTE GREŠKE

### Greška: "fatal: not a git repository"
**Rješenje:** Provjeri da li si u pravom direktoriju
```bash
cd devops-project-app
pwd  # Trebao bi pokazati put do projekta
```

### Greška: "error: src refspec main does not match any"
**Rješenje:** Prvo kreiraj commit
```bash
git add .
git commit -m "Initial commit"
```

### Greška: "Permission denied (publickey)"
**Rješenje:** Koristi HTTPS umjesto SSH
```bash
git remote set-url origin https://github.com/TVOJE_KORISNICKO_IME/devops-project-app.git
git push -u origin main
```

### GitHub traži lozinku
**Ako GitHub traži lozinku umjesto SSH ključa:**
- Koristi HTTPS URL (što već radiš)
- Za lozinku koristi GitHub Personal Access Token (vidi dolje)

---

## GitHub Personal Access Token (ako trebaš)

Ako GitHub ne prihvaća lozinku:

1. Na GitHub-u: Settings → Developer settings → Personal access tokens
2. Klikni "Generate new token"
3. Unesi:
   - Note: "DevOps Project"
   - Expiration: 90 days
   - Scopes: checkaj "repo"
4. Klikni "Generate token"
5. Kopiraj token (vidiš ga samo jednom!)
6. Kada GitHub traži lozinku, zalijepи token umjesto lozinke

---

## QUICK SUMMARY

| Korak | Komanda | Gdje |
|------|---------|------|
| 1 | Kreiraj račun | https://github.com/signup |
| 2 | Kreiraj repozitorij | https://github.com/new |
| 3 | Konfiguraj Git | `git config --global user.name "..."` |
| 4 | Init Git | `git init` |
| 5 | Dodaj datoteke | `git add .` |
| 6 | Kreiraj commit | `git commit -m "..."` |
| 7 | Dodaj remote | `git remote add origin https://...` |
| 8 | Pushaj | `git push -u origin main` |
| 9 | Provjeri | https://github.com/KORISNICKO_IME/devops-project-app |
| 10 | Pošalji profesoru | Pošalji link profesoru |

---

## GOTOV! 🎉

Kada vidiš sve datoteke na GitHub-u, projekt je uspješno učitan!

Sada pošalji profesoru link i gotovo je!
