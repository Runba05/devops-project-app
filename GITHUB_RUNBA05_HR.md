# GitHub Setup - Tvoje točne komande

## KORAK 1: Kreiraj novi repozitorij na GitHub-u

1. Logiraj se na https://github.com
2. Klikni "+" u gornjem desnom kutu
3. Odaberi "New repository"
4. Unesi:
   - **Repository name:** `devops-project-app`
   - **Description:** "Sigurna platforma za prodaju karata - DevOps/DevSecOps projekt"
   - **Public** ← MORA BITI JAVNA
   - Ostalo ostavi kao jest
5. Klikni "Create repository"

**Rezultat:** https://github.com/Runba05/devops-project-app

---

## KORAK 2: Konfiguriraj Git (prvi put)

```bash
git config --global user.name "Matej Barun"
git config --global user.email "barun.matej05@gmail.com"
```

---

## KORAK 3: Pushaj projekt (TVOJE TOČNE KOMANDE)

Kopija/paste ove komande u terminal/PowerShell:

```bash
git init
git add .
git commit -m "Inicijalni commit: Sigurna platforma za prodaju karata - DevOps/DevSecOps projekt"
git branch -M main
git remote add origin https://github.com/Runba05/devops-project-app.git
git push -u origin main
```

---

## KORAK 4: Provjera

Kada su sve komande gotove, idi na:

**https://github.com/Runba05/devops-project-app**

Trebao bi vidjeti sve datoteke:
- docker-compose.yml ✅
- k8s/base/ folder ✅
- docs/ folder ✅
- Dockerfiles ✅
- README.md i README_HR.md ✅
- itd.

---

## KORAK 5: Pošalji profesoru

Pošalji ovaj link:

```
https://github.com/Runba05/devops-project-app
```

---

## Primjer poruke za profesora

```
Predmet: Predaja DevOps projekta - Sigurna platforma za prodaju karata

Cijenjeni profesore,

Predajem svoj projekt za kolegij "Uvod u DevOps - DevSecOps":

PROJEKT: Sigurna platforma za prodaju karata
GitHub: https://github.com/Runba05/devops-project-app

SADRŽAJ:
✓ Dio 1: Lokalni razvojni stack (Docker Compose)
  - docker-compose.yml (5 servisa)
  - Dockerfiles (multi-stage, non-root)
  - README_HR.md - Lokalni setup

✓ Dio 2: Produkcijska Kubernetes implementacija
  - deployment.yaml (16 KB - svi resursi)
  - rbac.yaml (sigurnosne kontrole)
  - ingress-and-netpolicy.yaml (mrežne politike)
  - PRODUCTION_DEPLOYMENT_HR.md - Setup i runbook-ovi

✓ Sigurnost i DevSecOps
  - Trivy skeniranje
  - RBAC i najmanja razina pristupa
  - Mrežne politike
  - Multi-stage gradnje
  - Non-root korisnici

✓ CI/CD automatizacija
  - GitHub Actions cjevovod

✓ Dokumentacija
  - README_HR.md (lokalni razvoj)
  - SAZETAK_PROJEKTA_HR.md (pregled)
  - PRODUCTION_DEPLOYMENT_HR.md (produkcija)
  - TROUBLESHOOTING.md (rješenja)

GDJE POČETI:
1. Pročitaj README_HR.md
2. Vidi SAZETAK_PROJEKTA_HR.md za pregled
3. Pogledaj PRODUCTION_DEPLOYMENT_HR.md za detalje

ZA TESTIRANJE:
- Lokalno: docker compose up
- Kubernetes: kubectl apply -f k8s/base/

Hvala,
Matej Barun
```

---

## GOTOVO! 🎉

Kada sve komande pokreneš, projekt je na GitHub-u!

Tada samo pošalji profesoru link: **https://github.com/Runba05/devops-project-app**
