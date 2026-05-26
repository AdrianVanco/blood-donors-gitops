# Darcovia krvi — semestrálny projekt WAC

Aplikácia pre evidenciu darcov krvi a rezerváciu termínov odberov, integrovaná
do spoločného portálu „WAC Hospital" ako microfrontend.

## Členovia tímu

- Adrián Vančo

## Názov a identita aplikácie na spoločnom klastri

- **Názov v portáli (dlaždica):** Darcovia krvi
- **Web component / microfrontend:** `<cv2xvancoa-blood-donors>`
- **Cesta v portáli:** `/cv2xvancoa-blood-donors`
- **Kubernetes namespace:** `cv2xvancoa`
- **Deploymenty/služby:** `cv2xvancoa-blood-donors-ufe` (frontend), `cv2xvancoa-blood-donors-webapi` (backend)
- **Docker images:** `xvancoa/cv2xvancoa-blood-donors-ufe`, `xvancoa/cv2xvancoa-blood-donors-webapi`

## Repozitáre

- `blood-donors-ufe` — frontend (Stencil microfrontend)
- `blood-donors-webapi` — backend (Go + MongoDB)
- `blood-donors-gitops` — GitOps manifesty (Flux, Envoy Gateway, Dex, OPA, polyfea)

## Funkcionalita v skratke

- **Pracovník:** zoznam darcov s filtrami, stránkovaním a prepínačom typu odberu
  (krv/plazma), editor na vytváranie/úpravu/mazanie darcov a správu stavov termínov.
- **Darca:** „Môj účet" s vlastnými údajmi a históriou termínov; rezervácia termínu
  cez kalendár; zrušenie vlastnej rezervácie priamo v „Môj účet".
- **Pravidlá rezervácií (NTS SR):** odstup krv 70 dní / plazma 14 dní, ročný limit
  krvi (ženy 3×, muži 4×), kontrola spôsobilosti; darca môže mať súčasne len jednu
  prebiehajúcu rezerváciu (na novú musí pôvodnú zrušiť).
- **Autentifikácia a autorizácia:** prihlásenie cez OIDC/Dex, rozdelenie funkcií
  podľa role darca/pracovník vynútené v UI, na gateway (OPA) aj vo webapi; odhlásenie
  cez gateway.

> Demo dáta (darcovia) sú **fiktívne a anonymizované** — slúžia len na ukážku.

## Pokyny pre cvičiacich (špecifiká fungovania)

### Role a ich prideľovanie
Aplikácia rozlišuje dve role:
- **pracovník** — spravuje darcov (zoznam, vytváranie, úprava a mazanie záznamov);
- **darca** — vidí len vlastný účet a rezerváciu termínov.

Rola sa odvodzuje z **emailu** prihláseného používateľa. Pracovníci sú definovaní
zoznamom emailov, ktorý musí byť **zhodný na troch miestach**:
- OPA politika: `infrastructure/opa-plugin/params/policy.rego` (množina `worker_emails`),
- frontend: `blood-donors-ufe/src/global/auth.ts` (`WORKER_EMAILS`),
- backend: `blood-donors-webapi/internal/blood_donors/auth.go` (`workerEmails`).

Aktuálne je ako pracovník nastavený email `xvancoa@stuba.sk`. Každý ďalší
prihlásený používateľ je automaticky **darca**.

> Pracovník nie je darca — nemá „Môj účet" ani „Rezerváciu". Ak chce pracovník
> sám darovať, prihlási sa svojím osobným (darcovským) účtom.

### Ako otestovať jednotlivé role

Rola je viazaná na **email** v OIDC tokene, takže jeden účet = jedna rola.

- **V klastri (prihlásenie cez Dex/GitHub):**
  - *Pracovník* — ako pracovníci sú vopred zaradené aj emaily cvičiacich
    (`qunger@stuba.sk`, `qmicuch@stuba.sk`, `qsevcikm@stuba.sk`,
    `qhudakm1@stuba.sk`) popri `xvancoa@stuba.sk`. Po prihlásení ktorýmkoľvek
    z nich uvidíte správu darcov (zoznam, editor, vytváranie/mazanie). Zoznam je
    v `worker_emails` (`policy.rego`) a `WORKER_EMAILS` (`auth.ts`).
  - *Darca* — prihláste sa účtom, ktorý nie je v zozname pracovníkov. Demo
    darcovia sú fiktívni (anonymizované údaje), takže účet bez vlastného záznamu
    uvidí neutrálny stav „nemáte darcovský záznam". Plný darcovský profil
    najrýchlejšie uvidíte lokálne cez `?role=darca`, alebo nech pracovník vytvorí
    darcu s vaším emailom.
  - Pozn.: keďže rola = email, jeden účet zobrazuje vždy len jednu rolu.
    Pracovník nemá „Môj účet"/„Rezerváciu" (to potvrdzuje funkčnosť rozdelenia).
- **Lokálne bez prihlásenia (mock / FE+BE):** rolu prepnete priamo v URL —
  `…/blood-donors/?role=pracovnik` alebo `?role=darca`. (V klastri sa `?role`
  ignoruje, lebo rolu určuje token.)

### Prihlásenie a odhlásenie
- Prihlásenie zabezpečuje **Dex** (GitHub konektor) cez **Envoy Gateway**
  (`SecurityPolicy` na úrovni brány). Aplikácia nikdy nepracuje s heslom.
- Identita (email/meno) sa na frontende číta z ID token cookie
  `wac-hospital-id-token`; na backende z hlavičiek `x-forwarded-email` /
  `x-forwarded-roles`, ktoré dopĺňa gateway a OPA.
- Odhlásenie: tlačidlo „Odhlásiť" → `logoutPath` `/fea/logout`, kde Envoy vyčistí
  OIDC cookies. **Pozn.:** vyčistí sa len session na úrovni gateway; session
  u GitHubu/Dexu ostáva, takže ďalší prístup môže prihlásiť späť bez hesla.

### Registrácia darcu
- Účet (identita) pochádza z GitHubu cez Dex — v aplikácii sa nevytvára.
- **Darcovský záznam** vytvára pracovník v editore (akcia povolená len pracovníkovi).
- Prihlásený darca, ktorý ešte nemá vlastný záznam, uvidí neutrálnu hlášku
  „nemáte darcovský záznam, kontaktujte pracovníka" (nezobrazí sa cudzí profil).

### Vynútenie role na backende (defense-in-depth)
Okrem UI je rola vynútená aj vo webapi. Pracovníka rozpozná z role `pracovnik`
v hlavičke `x-forwarded-roles` (od OPA) **alebo** z emailu `x-forwarded-email`
v zozname `workerEmails` (fallback — na zdieľanom klastri nemusí spoločná OPA
posielať aplikačnú rolu, email z tokenu však áno):
- vytvorenie a mazanie darcu — len **pracovník**,
- úprava darcu — **pracovník**, alebo **darca upravujúci vlastný záznam**
  (zhoda `x-forwarded-email` s emailom darcu).

Pri priamom prístupe mimo gateway (lokálny vývoj) sa autorizácia nevynucuje —
v produkcii je webapi dostupné výhradne cez gateway, ktorý neautentifikované
požiadavky odmietne na okraji.

### Rezervácie termínov
- Darca môže mať súčasne len **jednu prebiehajúcu rezerváciu**. Kým ju má, kalendár
  ďalšiu rezerváciu nedovolí (a zobrazí upozornenie). Pre novú musí pôvodnú najprv
  **zrušiť v sekcii „Môj účet"**.
- Rezervácia sa ukladá ako termín darcu (stav „Rezervácia dokončená") a je viditeľná
  aj pracovníkovi v editore darcu.

## Lokálne spustenie (mimo klastra)

1. **Mock** (len UI): v `blood-donors-ufe` spustiť `npm start` (FE na :3333, mock
   API na :5000). Zápisy sa neukladajú (mock je bezstavový). Rola sa dá prepnúť
   cez URL: `…/blood-donors/?role=pracovnik` alebo `?role=darca`.
2. **FE + Go webapi + MongoDB**: v `blood-donors-webapi/deployments/docker-compose`
   spustiť `docker compose up -d` (Mongo + mongo-express), naštartovať webapi
   `go run ./cmd/blood-donors-api-service` (port :8080), a v
   `blood-donors-ufe/src/index.html` prepnúť `api-base` na `http://localhost:8080/api`,
   potom `npm run start:app`. Zápisy sa ukladajú do MongoDB.
3. **Klaster (GitOps)**: nasadenie cez `scripts/developer-deploy.ps1`. Až tu reálne
   fungujú prihlásenie cez Dex, OPA autorizácia a odhlásenie.
