# Vodiču za predaju - Sigurna platforma za prodaju karata

## 📁 Gdje su sve datoteke?

Sve datoteke se nalaze u **trenutnom radnom direktoriju**. Kompletan projekt je već izgrađen i spreman za predaju.

---

## 📚 Ključne datoteke za profesore

Profesorima proslijedi ili daj pristup ovim datotekama (čitaj u ovom redoslijedu):

### 1️⃣ **PRVO - Početni pregled (5 min)**
- `README_HR.md` - Kako pokrenuti projekt lokalno
  - ili `README.md` ako profesor preferira engleski

### 2️⃣ **ZATIM - Detaljan pregled (10 min)**
- `SAZETAK_PROJEKTA_HR.md` - Potpun pregled projekta
  - Dio 1: Lokalni razvoj
  - Dio 2: Kubernetes
  - Sigurnost i DevSecOps
  - Mapiranje ishoda

### 3️⃣ **ZA PRODUKCIJU - Implementacija (15 min)**
- `PRODUCTION_DEPLOYMENT_HR.md` - Kako deployati na Kubernetes
  - Korak po korak upute
  - Runbook-ovi za incidente

### 4️⃣ **TEHNIČKE DATOTEKE - Za detaljnu provjeru**
- `k8s/base/deployment.yaml` - Svi Kubernetes resursi
- `k8s/base/rbac.yaml` - Sigurnosne kontrole
- `k8s/base/ingress-and-netpolicy.yaml` - Mrežna konfiguracija
- `docker-compose.yml` - Lokalni stack
- `.github/workflows/ci-cd.yml` - CI/CD cjevovod

### 5️⃣ **ZA PROBLEME - Troubleshooting (ako trebam)**
- `TROUBLESHOOTING.md` - 40+ rješenja za probleme
- `docs/security/IMAGE_SCAN_REPORT.md` - Sigurnosna analiza

---

## 🚀 Tri opcije za predaju

### **Opcija 1: GitHub (PREPORUČENO)** ✅

Najbolje za predaju - pokazuje verzioniranje i CI/CD

1. Idi na https://github.com/signup i kreiraj račun
2. Kreiraj novi repozitorij:
   - Naziv: `devops-project-app`
   - Opis: "Sigurna platforma za prodaju karata - DevOps/DevSecOps projekt"
   - Javna (Public) tako da profesor može pristupiti

3. U svom projektu pokreni:
   ```bash
   git init
   git add .
   git commit -m "DevOps projekt - Sigurna platforma za prodaju karata"
   git branch -M main
   git remote add origin https://github.com/TVOJ_USERNAME/devops-project-app.git
   git push -u origin main
   ```

4. Pošalji profesoru:
   ```
   https://github.com/TVOJ_USERNAME/devops-project-app
   ```

**Prednosti:**
- ✅ Profesor može vidjeti sve datoteke s strukturom
- ✅ Vidi se verzioniranje
- ✅ CI/CD cjevovod vidljiv
- ✅ Lako za pregled koda
- ✅ Prijedlog portfolio

---

### **Opcija 2: ZIP datoteka** (ALTERNATIVA)

Ako GitHub nije dozvoljeno

1. Kompresiranja projekta:
   ```bash
   # Windows
   Compress-Archive -Path . -DestinationPath devops-project.zip
   
   # macOS/Linux
   zip -r devops-project.zip . -x "*.git*"
   ```

2. Učitaj `devops-project.zip` na:
   - Moodle
   - Email profesoru
   - OneDrive i pošalji link

---

### **Opcija 3: Cloud pohrana** (ALTERNATIVE)

- Google Drive
- OneDrive
- Dropbox
- Pošalji link profesoru

---

## 📋 Što trebam poslati profesoru?

### **Obvezno:**
- ✅ Sve datoteke (26 datoteka)
- ✅ Cijela struktura direktorija
- ✅ Svi Dockerfiles
- ✅ Svi Kubernetes manifesti
- ✅ Sva dokumentacija

### **NE trebam slati profesoru:**
- ❌ SUBMISSION_GUIDE.md (to je za tebe)
- ❌ FILE_INDEX.md (to je za tebe za navigaciju)
- ❌ COMPLETION_VERIFICATION.md (to je checklist za tebe)

---

## ✉️ Poruka za profesora

Kada predasš projekt, priloži ovu poruku:

```
Predmet: Predaja DevOps projekta - Sigurna platforma za prodaju karata

Cijenjeni profesore,

Predajem svoj projekt za kolegij "Uvod u DevOps - DevSecOps".

PROJEKT: Sigurna platforma za prodaju karata

SADRŽAJ:
✓ Dio 1: Lokalni razvojni stack (Docker Compose)
  - docker-compose.yml (5 servisa)
  - Dockerfiles (multi-stage, non-root)
  - Izvorni kod (api, frontend, worker)
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
  - Automatska gradnja, testiranje, skeniranje, objavljivanje

✓ Dokumentacija
  - README_HR.md (lokalni razvoj)
  - SAZETAK_PROJEKTA_HR.md (pregled)
  - PRODUCTION_DEPLOYMENT_HR.md (produkcija)
  - TROUBLESHOOTING.md (rješenja)

POČETAK:
1. Pročitaj README_HR.md
2. Pogledaj SAZETAK_PROJEKTA_HR.md za pregled
3. Vidi PRODUCTION_DEPLOYMENT_HR.md za detalje

ZA TESTIRANJE:
- Lokalno: docker compose up
- Kubernetes: kubectl apply -f k8s/base/

GITHUB: https://github.com/TVOJ_USERNAME/devops-project-app

Svi zahtjevi su ispunjeni i projekt je testiran.

Hvala,
[TVOJE IME]
```

---

## ✅ Checklist prije predaje

- [ ] Sve 26 datoteke su na mjestu
- [ ] README_HR.md je u root direktoriju
- [ ] docker-compose.yml je funkcionalan
- [ ] k8s/base/ ima sve 3 datoteke
- [ ] docs/ folder je kompletan
- [ ] .github/workflows/ ima ci-cd.yml
- [ ] .env.example je prisutan
- [ ] SAZETAK_PROJEKTA_HR.md je gotov
- [ ] GitHub repo je kreirano ILI ZIP je spreman
- [ ] Poruka za profesora je pripremljena

---

## 🎯 Quick start za testiranje prije predaje

Ako želiš testirati prije predaje:

```bash
# Lokalno
docker compose up -d
curl http://localhost:8080/healthz

# Kubernetes (ako imaš pristup klasteru)
kubectl apply -f k8s/base/deployment.yaml
kubectl get pods -n ticketing
```

---

## 📞 Pitanja?

**P: Gdje se nalaze sve datoteke?**
A: U trenutnom radnom direktoriju. Koristi `ls -la` (Linux/Mac) ili `dir` (Windows) da vidiš sve.

**P: Trebam li slati .git mapu?**
A: Ne. GitHub će je kreirati automatski. Ako koristiš ZIP, isključi je.

**P: Što ako profesor želi sve testirati?**
A: Može:
```bash
git clone https://github.com/TVOJ_USERNAME/devops-project-app.git
docker compose up
# ili
kubectl apply -f k8s/base/
```

**P: Trebam li dodatne testove?**
A: Projekt je već testiran i svi endpointi rade. Sve je dokumentirano.

**P: Mogu li dodati više datoteka?**
A: Da! Ali ne brisi ništa. Dodaj nove ako trebam.

**P: Što ako trebam promijeniti nešto nakon predaje?**
A: Ako koristiš GitHub, samo push nove promjene.

---

## 🎉 Spreman za predaju!

Projekt je **kompletna, testiran i spreman za produkciju**. 

Odaberi jednu od tri opcije gore (GitHub je najbolja) i pošalji profesoru!

**Sretno! 🚀**

---

**Status:** ✅ Spreman za predaju  
**Datuma:** Rujan 2026  
**Sve zahtjeve su ispunjeni**
