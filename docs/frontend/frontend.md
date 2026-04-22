# Frontend dokumentáció

## UFO és Paranormális Bejelentő Rendszer – Angular SPA

---

## Tartalomjegyzék

1. [Áttekintés](#1-áttekintés)
2. [Technológiai stack](#2-technológiai-stack)
3. [Projektstruktúra](#3-projektstruktúra)
4. [Környezeti konfiguráció](#4-környezeti-konfiguráció)
5. [Útvonalak és oldalak](#5-útvonalak-és-oldalak)
6. [Komponensek](#6-komponensek)
7. [Szolgáltatások](#7-szolgáltatások)
8. [Guard-ok és interceptor](#8-guard-ok-és-interceptor)
9. [Stílus rendszer](#9-stílus-rendszer)

---

## 1. Áttekintés

A frontend egy **Angular 19** alapú single-page application (SPA), amely a Laravel REST API-val kommunikál. Az alkalmazás paranormális jelenségek és UFO-észlelések böngészésére, bejelentésére és moderálására szolgál.

Az összes oldal **lazy-loaded standalone component** – nincs közös modul, minden oldal önállóan töltődik be, csak akkor amikor szükséges. A változásdetektáció mindenhol `OnPush` stratégiával működik a jobb teljesítmény érdekében.

**Főbb jellemzők:**
- Bejelentések listázása, szűrése és rendezése
- Bejelentés létrehozása, szerkesztése, képfeltöltéssel
- Térkép alapú helyszín megjelenítés és kiválasztás (Leaflet.js)
- Upvote/downvote szavazási rendszer
- Regisztráció, bejelentkezés, profil szerkesztése
- Admin felület: moderálás, felhasználókezelés, kategóriakezelés
- Sci-fi stílusú, sötét témájú felhasználói felület

---

## 2. Technológiai stack

| Komponens          | Verzió / Leírás                                      |
|--------------------|------------------------------------------------------|
| Angular            | 19 (standalone components, signals)                  |
| TypeScript         | 5.x                                                  |
| Bootstrap          | 5.3.3 (CDN, scss változók felülírva)                 |
| Bootstrap Icons    | 1.11.3 (CDN)                                         |
| Leaflet.js         | Térkép megjelenítés                                  |
| RxJS               | HTTP kommunikáció, debounce szűrés                   |
| Angular Signals    | Reaktív állapotkezelés (`AuthService.currentUser`)   |
| Google Fonts       | Orbitron, Share Tech Mono, Inter                     |

---

## 3. Projektstruktúra

```
frontend/src/
├── index.html                  # App belépési pont, Bootstrap CDN linkek
├── main.ts                     # bootstrapApplication() hívás
├── styles.css                  # Globális sci-fi téma, Bootstrap változók felülírása
└── app/
    ├── app.ts                  # Gyökér komponens (Navbar + router-outlet)
    ├── app.html                # <router-outlet />
    ├── app.config.ts           # provideRouter, provideHttpClient, authInterceptor
    ├── app.routes.ts           # Összes útvonal definíciója
    ├── pages/
    │   ├── report-list/        # Főoldal – bejelentések listája
    │   ├── report-detail/      # Bejelentés részletei
    │   ├── report-form/        # Létrehozás és szerkesztés (közös oldal)
    │   ├── statistics/         # Statisztikák
    │   ├── profile/            # Saját profil és bejelentések
    │   ├── login/              # Bejelentkezés
    │   ├── register/           # Regisztráció
    │   ├── admin-reports/      # Admin: bejelentések moderálása
    │   ├── admin-users/        # Admin: felhasználókezelés
    │   └── admin-categories/   # Admin: kategória CRUD
    ├── components/
    │   ├── navbar/             # Navigációs sáv
    │   └── map/                # Leaflet térkép komponens
    ├── services/
    │   ├── auth.ts             # Regisztráció, bejelentkezés, token, currentUser signal
    │   ├── report.ts           # CRUD, képfeltöltés, szűrés
    │   ├── vote.ts             # Upvote/downvote
    │   ├── category.ts         # Kategória lekérdezés és CRUD (admin)
    │   └── admin.ts            # Admin: bejelentés, felhasználó kezelés
    ├── guards/
    │   ├── auth-guard.ts       # Bejelentkezés ellenőrzése
    │   └── admin-guard.ts      # Admin szerepkör ellenőrzése
    ├── interceptors/
    │   └── auth-interceptor.ts # Bearer token csatolása minden HTTP kéréshez
    └── environments/
        └── environment.ts      # API és storage URL konfiguráció
```

---

## 4. Környezeti konfiguráció

```typescript
// src/environments/environment.ts
export const environment = {
  production: false,
  apiUrl:     'http://localhost:8000/api',
  storageUrl: 'http://localhost:8000/storage',
};
```

A képek URL-je: `storageUrl + '/' + image_path`, pl. `http://localhost:8000/storage/report_images/abc123.jpg`

---

## 5. Útvonalak és oldalak

Az összes útvonal lazy-loaded – az oldal kódja csak az első látogatáskor töltődik le.

| Útvonal              | Komponens          | Guard        | Leírás                                          |
|----------------------|--------------------|--------------|-------------------------------------------------|
| `/`                  | `ReportList`       | —            | Bejelentések listája, slider, szűrés, lapozás   |
| `/reports/:id`       | `ReportDetail`     | —            | Bejelentés részletei, térkép, szavazás, képek   |
| `/reports/create`    | `ReportForm`       | `authGuard`  | Új bejelentés létrehozása                       |
| `/reports/:id/edit`  | `ReportForm`       | `authGuard`  | Meglévő bejelentés szerkesztése                 |
| `/statistics`        | `Statistics`       | —            | Statisztikák, top lista, kategória bontás       |
| `/profile`           | `Profile`          | `authGuard`  | Saját adatok, jelszó, saját bejelentések        |
| `/login`             | `Login`            | —            | Bejelentkezési űrlap                            |
| `/register`          | `Register`         | —            | Regisztrációs űrlap                             |
| `/admin/reports`     | `AdminReports`     | `adminGuard` | Státuszszűrés, jóváhagyás, elutasítás, törlés   |
| `/admin/users`       | `AdminUsers`       | `adminGuard` | Felhasználók listázása, keresés, tiltás         |
| `/admin/categories`  | `AdminCategories`  | `adminGuard` | Kategória hozzáadás, szerkesztés, törlés        |
| `**`                 | —                  | —            | Ismeretlen útvonal → `/`                        |

---

## 6. Komponensek

### ReportList (főoldal)

<img width="1920" height="1080" alt="Képernyőfelvétel (629)" src="https://github.com/user-attachments/assets/f6c6feca-49b6-426a-9b65-efa9dffe5882" />

A főoldalon három fő rész található:

**1. Slider** – A top 3 leghitelesebb bejelentés auto-lejátszós csúszkán jelenik meg (5 másodpercenként vált, manuálisan is léptethető). A hitelességi pontszám = upvote − downvote.

**2. Szűrő és rendező panel** – Kategória, dátum intervallum és rendezési szempont választható.

| Rendezési lehetőség              | Paraméter                        |
|----------------------------------|----------------------------------|
| Feltöltés dátuma (újabb elöl)    | `created_at` / `desc` (alapért.) |
| Feltöltés dátuma (régebbi elöl)  | `created_at` / `asc`             |
| Esemény dátuma (újabb elöl)      | `date` / `desc`                  |
| Esemény dátuma (régebbi elöl)    | `date` / `asc`                   |
| Cím (A → Z)                      | `title` / `asc`                  |
| Cím (Z → A)                      | `title` / `desc`                 |
| Hitelesség (nagyobb elöl)        | `credibility` / `desc`           |
| Hitelesség (kisebb elöl)         | `credibility` / `asc`            |

**3. Lapozás** – A bejelentések oldalanként 9 kártyán jelennek meg. Lapváltáskor az oldal tetejére ugrik.

<img width="1920" height="1080" alt="Képernyőfelvétel (636)" src="https://github.com/user-attachments/assets/b42eeec7-1cdf-4965-b367-0948625c7e4c" />

---

### ReportDetail (bejelentés részletei)

<img width="1920" height="1080" alt="Képernyőfelvétel (634)" src="https://github.com/user-attachments/assets/585c24e8-eb1e-4393-b447-95cecb15a9f6" />

- Megjeleníti a bejelentés összes adatát (cím, leírás, kategória, dátum, tanúk száma, státusz, feltöltő neve)
- Ha van koordináta, Leaflet térképen mutatja a helyszínt (readonly mód)
- **Hitelesség szavazás**: bejelentkezett felhasználó upvote/downvote-olhat; azonos szavazat ismételt leadása visszavonja azt; ellentétes szavazat felülírja
- **Képek**: bélyegképek gallériaként jelennek meg, lightbox nézettel (nyíl, ESC gomb)
- **Jogosultság alapján** megjelenik a szerkesztés és törlés gomb (saját bejelentésnél, vagy admin esetén), illetve képfeltöltés lehetősége

---

### ReportForm (bejelentés létrehozása / szerkesztése)

<img width="1920" height="1080" alt="Képernyőfelvétel (632)" src="https://github.com/user-attachments/assets/826d93a3-3867-4751-a52c-4c788ec05fe0" />
<img width="1920" height="1080" alt="Képernyőfelvétel (635)" src="https://github.com/user-attachments/assets/e85da849-6c78-4883-b322-9726a32a85d1" />

Ugyanaz a komponens kezeli az új bejelentést (`/reports/create`) és a szerkesztést (`/reports/:id/edit`). Az `isEdit` flag alapján dől el melyik módban fut.

**Mezők:**

| Mező          | Típus          | Kötelező | Megjegyzés                              |
|---------------|----------------|----------|-----------------------------------------|
| Kategória     | select         | igen     | API-ból töltődik be                     |
| Cím           | text           | igen     | max 255 karakter                        |
| Leírás        | textarea       | igen     | –                                       |
| Esemény dátuma| date           | igen     | Nem lehet jövőbeli                      |
| Tanúk száma   | number         | nem      | Egész szám, minimum 1                   |
| Helyszín      | Leaflet térkép | nem      | Kattintásra beállítja a koordinátákat   |
| Képek         | file input     | nem      | Max 10 db, max 5 MB/kép, előnézet       |

Szerkesztésnél a meglévő képek listában jelennek meg, egyenként törölhetők. Új képek is feltölthetők szerkesztés közben.

---

### Statistics (statisztikák)

<img width="1920" height="1080" alt="Képernyőfelvétel (631)" src="https://github.com/user-attachments/assets/57a381d1-7da8-4a19-b658-a2c2f79b3cd6" />

- **Összesítők**: jóváhagyott bejelentések száma, regisztrált felhasználók, leadott szavazatok
- **Státusz bontás**: pending / approved / rejected darabszám badge-ekkel
- **Kategória bontás**: vízszintes progress bar, a legnagyobb értéket 100%-nak veszi, a többi ahhoz arányított
- **Top 5 leghitelesebb**: hitelességi pontszám szerint rendezve, bejelentésre mutató linkkel
- **Legutóbbi 5**: létrehozás dátuma szerint, bejelentésre mutató linkkel

---

### Profile (profil)

<img width="1920" height="1080" alt="Képernyőfelvétel (633)" src="https://github.com/user-attachments/assets/c4a63152-49f1-4de5-a2e2-e82cdd048d75" />

- Megjeleníti a bejelentkezett felhasználó nevét, e-mail-jét és bejelentéseinek számát
- Szerkeszthető: név, e-mail, jelszó
- Listázza a felhasználó összes saját bejelentését státuszjelzőkkel és linkekkel

---

### Login / Register

<img width="1920" height="1080" alt="Képernyőfelvétel (637)" src="https://github.com/user-attachments/assets/f4d90220-cb9c-4ee6-b8d7-01680a47a223" />
<img width="1920" height="1080" alt="Képernyőfelvétel (638)" src="https://github.com/user-attachments/assets/99059bed-5236-43f1-9318-024421a2d03b" />

Egyszerű űrlapok templateDriven FormsModule-lal. A backend validációs hibákat mezőnként kapja meg az alkalmazás, összefűzve jeleníti meg egy hibaüzenetben. Sikeres bejelentkezés / regisztráció után a főoldalra navigál.

---

### AdminReports

<img width="1920" height="1080" alt="Képernyőfelvétel (640)" src="https://github.com/user-attachments/assets/49cba92f-fdd0-4965-ae35-29539245bcf7" />

- Státusz szerint szűrhető lista (összes / várakozik / jóváhagyva / elutasítva)
- Minden sornál: jóváhagyás, elutasítás, törlés (megerősítő confirm dialóggal)
- Bejelentésre kattintva megnyílik a részletes nézet

---

### AdminUsers

<img width="1920" height="1080" alt="Képernyőfelvétel (641)" src="https://github.com/user-attachments/assets/00b2cb79-d0c7-4b6a-bb18-177d5229af56" />

- Felhasználók táblázata regisztráció dátumával, bejelentésszámával és szerepkörrel
- Valós idejű keresés névre és e-mailre (az API-ra küldött `?search=` paraméterrel)
- Tiltás / tiltás feloldása gomb soronként (tiltott sor pirossal kiemelve)

---

### AdminCategories

<img width="1920" height="1080" alt="Képernyőfelvétel (642)" src="https://github.com/user-attachments/assets/35f07202-f303-4d8d-8457-3d4b45bb3882" />

- Kategória hozzáadása és szerkesztése egyetlen űrlapon (szerkesztésnél az űrlap előtöltődik)
- Törlés megerősítő dialóggal
- A lista rögtön frissül minden módosítás után

---

### Navbar (komponens)

<img width="1920" height="1080" alt="Képernyőfelvétel (639)" src="https://github.com/user-attachments/assets/18db2203-bf74-443a-846a-b944714ab83c" />

Állapotfüggő megjelenítés:

| Bejelentkezve | Admin | Megjelenő elemek                                        |
|---------------|-------|----------------------------------------------------------|
| nem           | nem   | Bejelentések, Statisztikák, Bejelentkezés, Regisztráció  |
| igen          | nem   | + Új bejelentés, Profil, Kijelentkezés                   |
| igen          | igen  | + Admin menü (Bejelentések, Felhasználók, Kategóriák)    |

---

### Map (komponens)

Leaflet.js alapú térkép, két módban:

| Mód              | Input             | Output       | Leírás                                          |
|------------------|-------------------|--------------|-------------------------------------------------|
| Csak megjelenítő | `[lat]`, `[lng]`  | —            | Koordináta alapján mutatja a helyszínt, readonly|
| Interaktív       | `[clickable]=true`| `(mapClick)` | Kattintáskor visszaadja `[lat, lng]` tömböt    |

A `height` input (pl. `"300px"`) dinamikusan állítja a doboz magasságát.

```html
<!-- Megjelenítő mód -->
<app-map [lat]="report.latitude" [lng]="report.longitude" [clickable]="false" height="300px" />

<!-- Interaktív mód -->
<app-map [clickable]="true" (mapClick)="onMapClick($event)" height="400px" />
```

A Leaflet OpenStreetMap csempéket használ. A `invalidateSize()` meghívása fontos, különben szürke csempék jelennek meg, ha a konténer mérete az inicializálás után változott.

---

## 7. Szolgáltatások

### AuthService

A bejelentkezett felhasználó állapotát Angular **Signal**-ban (`currentUser`) tárolja. Az adat a `sessionStorage`-ban marad, így oldal-újratöltés után sem vész el a munkamenet.

| Metódus        | Leírás                                                             |
|----------------|--------------------------------------------------------------------|
| `register()`   | POST `/api/register` → token és user mentése sessionStorage-ba    |
| `login()`      | POST `/api/login` → token és user mentése sessionStorage-ba       |
| `logout()`     | POST `/api/logout`, sessionStorage törlése, navigál `/login`-ra   |
| `getToken()`   | Visszaadja a tárolt Bearer tokent                                  |
| `isLoggedIn()` | `true` ha van token                                               |
| `isAdmin()`    | `true` ha `currentUser().role === 'admin'`                        |

---

### ReportService

| Metódus           | Végpont                              | Leírás                          |
|-------------------|--------------------------------------|---------------------------------|
| `getAll(filters)` | GET `/api/reports`                   | Szűrők query paraméterként      |
| `getOne(id)`      | GET `/api/reports/:id`               | –                               |
| `create(data)`    | POST `/api/reports`                  | –                               |
| `update(id,data)` | PUT `/api/reports/:id`               | –                               |
| `delete(id)`      | DELETE `/api/reports/:id`            | Soft delete                     |
| `getMapReports()` | GET `/api/map/reports`               | Koordinátás bejelentések        |
| `uploadImages()`  | POST `/api/reports/:id/images`       | `FormData` – `images[]` tömb    |

---

### VoteService

Egyetlen metódus: `vote(reportId, 'up' | 'down')` → POST `/api/reports/:id/vote`

Ugyanolyan szavazat = visszavonás. Ellentétes szavazat = felülírás. A válasz tartalmazza az aktuális upvote/downvote számlálókat.

---

### CategoryService

| Metódus        | Leírás                          |
|----------------|---------------------------------|
| `getAll()`     | GET `/api/categories`           |
| `create(data)` | POST `/api/admin/categories`    |
| `update(id)`   | PUT `/api/admin/categories/:id` |
| `delete(id)`   | DELETE `/api/admin/categories/:id` |

---

### AdminService

| Metódus              | Végpont                                   | Leírás                      |
|----------------------|-------------------------------------------|-----------------------------|
| `getReports(status)` | GET `/api/admin/reports?status=`          | Opcionális státuszszűrő     |
| `approveReport(id)`  | PUT `/api/admin/reports/:id/approve`      | –                           |
| `rejectReport(id)`   | PUT `/api/admin/reports/:id/reject`       | –                           |
| `deleteReport(id)`   | DELETE `/api/admin/reports/:id`           | –                           |
| `getUsers(search)`   | GET `/api/admin/users?search=`            | Opcionális keresőszó        |
| `banUser(id)`        | PUT `/api/admin/users/:id/ban`            | –                           |
| `unbanUser(id)`      | PUT `/api/admin/users/:id/unban`          | –                           |

---

## 8. Guard-ok és interceptor

### authGuard

Ellenőrzi, hogy van-e érvényes token a `sessionStorage`-ban. Ha nincs, `/login`-ra irányít.

```typescript
// auth-guard.ts
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  return auth.isLoggedIn() ? true : inject(Router).createUrlTree(['/login']);
};
```

### adminGuard

Ellenőrzi, hogy a bejelentkezett felhasználó adminisztrátor-e. Ha nem, `/`-re irányít.

```typescript
// admin-guard.ts
export const adminGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  return auth.isAdmin() ? true : inject(Router).createUrlTree(['/']);
};
```

### authInterceptor

Minden kimenő HTTP kéréshez automatikusan csatolja a Bearer tokent, ha az elérhető.

```typescript
// auth-interceptor.ts
const token = inject(AuthService).getToken();
if (token) {
  req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
}
return next(req);
```

---

## 9. Stílus rendszer

A globális stílus (`src/styles.css`) egy sötét sci-fi témát valósít meg Bootstrap változók felülírásával.

### CSS változók (`:root`)

| Változó         | Érték       | Felhasználás                          |
|-----------------|-------------|---------------------------------------|
| `--sf-bg`       | `#0b0e14`   | Oldal háttere                         |
| `--sf-surface`  | `#111620`   | Kártyák, táblázatok háttere           |
| `--sf-surface2` | `#161c2a`   | Form elemek, kártya fejléc háttere    |
| `--sf-border`   | `#1e2d45`   | Körvonalak, elválasztók               |
| `--sf-accent`   | `#3a8fc8`   | Kék kiemelőszín (gombok, linkek)      |
| `--sf-accent2`  | `#4db8c8`   | Élénkebb kék (hover, brand)           |
| `--sf-text`     | `#dde6f5`   | Fő szövegszín                         |
| `--sf-text-dim` | `#8a9dba`   | Halvány szöveg, labels                |

### Betűtípusok

- **Orbitron** – brand logó, statisztika számok (sci-fi megjelenés)
- **Share Tech Mono** – badge-ek, kategóriacímkék
- **Inter** – általános szöveg

### Bootstrap felülírás

A Bootstrap saját CSS változói (`--bs-body-bg`, `--bs-card-bg`, stb.) a `:root`-ban kerülnek felülírásra, így az összes Bootstrap komponens automatikusan a sci-fi témát használja, külön osztályok nélkül.

