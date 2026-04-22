# UFO és Paranormális Bejelentő Rendszer – Teljes Dokumentáció

**Készítők:**
- Backend: Péntek Patrik Sándor
- Frontend: Seletye Odett

**Vizsgaremek — 2026**

---

## Tartalomjegyzék

### Backend
1. [Áttekintés](#1-backend-áttekintés)
2. [Technológiai stack (backend)](#2-technológiai-stack-backend)
3. [Adatbázis struktúra](#3-adatbázis-struktúra)
4. [Modellek és kapcsolatok](#4-modellek-és-kapcsolatok)
5. [Jogosultságkezelés](#5-jogosultságkezelés)
6. [Middleware-ek](#6-middleware-ek)
7. [Form Request validáció](#7-form-request-validáció)
8. [API végpontok](#8-api-végpontok)
9. [Controllerek](#9-controllerek)
10. [Seederek](#10-seederek)
11. [Tesztek](#11-tesztek)

### Frontend
12. [Áttekintés](#12-frontend-áttekintés)
13. [Technológiai stack (frontend)](#13-technológiai-stack-frontend)
14. [Projektstruktúra](#14-projektstruktúra)
15. [Környezeti konfiguráció](#15-környezeti-konfiguráció)
16. [Útvonalak és oldalak](#16-útvonalak-és-oldalak)
17. [Komponensek](#17-komponensek)
18. [Szolgáltatások](#18-szolgáltatások)
19. [Guard-ok és interceptor](#19-guard-ok-és-interceptor)
20. [Stílus rendszer](#20-stílus-rendszer)

---

# BACKEND

---

## 1. Backend áttekintés

A backend egy **Laravel 12** alapú REST API, amely egy paranormális jelenségeket és UFO-észleléseket gyűjtő és moderáló rendszert valósít meg. A rendszerben bejelentkezés nélkül is böngészhető tartalom van, de bejelentések rögzítéséhez, szavazáshoz és adminisztrációhoz regisztráció és hitelesítés szükséges.

Az autentikáció **Laravel Sanctum** Bearer token alapon működik. Alap URL: `http://localhost:8000/api`

**Főbb funkciók:**
- Regisztráció és bejelentkezés Laravel Sanctum tokenes hitelesítéssel
- Paranormális bejelentések (report) létrehozása, szerkesztése, törlése
- Kategóriák kezelése (admin)
- Képfeltöltés bejelentésekhez
- Szavazás (upvote/downvote) a bejelentések hitelességére
- Admin moderálás: jóváhagyás, elutasítás, felhasználó tiltás
- Statisztikák publikus és admin szinten
- Térkép nézethez koordináta-alapú lekérdezés

---

## 2. Technológiai stack (backend)

| Komponens        | Verzió / Csomag              |
|------------------|------------------------------|
| PHP              | ^8.2                         |
| Laravel          | ^12.0                        |
| Hitelesítés      | Laravel Sanctum ^4.3         |
| Adatbázis        | MySQL (XAMPP)                |
| Tesztelés        | PHPUnit ^11.5                |
| Dev eszközök     | Laravel Pint, Laravel Pail   |

---

## 3. Adatbázis struktúra

### `users`

| Mező                | Típus                      | Leírás                                |
|---------------------|----------------------------|---------------------------------------|
| `id`                | bigint (PK)                | Egyedi azonosító                      |
| `name`              | string                     | Felhasználó neve                      |
| `email`             | string (unique)            | E-mail cím                            |
| `email_verified_at` | timestamp, nullable        | E-mail megerősítés ideje              |
| `password`          | string (hashed)            | Jelszó (bcrypt)                       |
| `role`              | enum: `user`, `admin`      | Szerepkör, alapértelmezett: `user`    |
| `is_banned`         | boolean                    | Tiltás állapota, default: `false`     |
| `remember_token`    | string, nullable           | Emlékezz rám token                    |
| `created_at`        | timestamp                  | Létrehozás ideje                      |
| `updated_at`        | timestamp                  | Módosítás ideje                       |
| `deleted_at`        | timestamp, nullable        | Soft delete ideje                     |

### `categories`

| Mező          | Típus               | Leírás                      |
|---------------|---------------------|-----------------------------|
| `id`          | bigint (PK)         | Egyedi azonosító             |
| `name`        | string (unique)     | Kategória neve               |
| `description` | text, nullable      | Leírás                      |
| `created_at`  | timestamp           | Létrehozás ideje             |
| `updated_at`  | timestamp           | Módosítás ideje              |
| `deleted_at`  | timestamp, nullable | Soft delete ideje            |

### `reports`

| Mező          | Típus                                   | Leírás                               |
|---------------|-----------------------------------------|--------------------------------------|
| `id`          | bigint (PK)                             | Egyedi azonosító                      |
| `user_id`     | FK → `users.id` (cascade delete)       | Bejelentő felhasználó                 |
| `category_id` | FK → `categories.id` (cascade delete)  | Kategória                             |
| `title`       | string                                  | Bejelentés címe                       |
| `description` | text                                    | Részletes leírás                      |
| `latitude`    | decimal(10,7), nullable                 | GPS szélességi fok                    |
| `longitude`   | decimal(10,7), nullable                 | GPS hosszúsági fok                    |
| `date`        | date                                    | Esemény dátuma                        |
| `witnesses`   | unsigned integer, default: 0            | Tanúk száma                           |
| `status`      | enum: `pending`, `approved`, `rejected` | Moderálás állapota (default: pending) |
| `created_at`  | timestamp                               | Létrehozás ideje                      |
| `updated_at`  | timestamp                               | Módosítás ideje                       |
| `deleted_at`  | timestamp, nullable                     | Soft delete ideje                     |

### `report_images`

| Mező         | Típus                               | Leírás                       |
|--------------|-------------------------------------|------------------------------|
| `id`         | bigint (PK)                         | Egyedi azonosító              |
| `report_id`  | FK → `reports.id` (cascade delete)  | Bejelentés azonosítója        |
| `image_path` | string                              | Kép relatív elérési útja      |
| `created_at` | timestamp, nullable                 | Feltöltés ideje               |
| `deleted_at` | timestamp, nullable                 | Soft delete ideje             |

### `votes`

| Mező        | Típus                               | Leírás                            |
|-------------|-------------------------------------|-----------------------------------|
| `id`        | bigint (PK)                         | Egyedi azonosító                   |
| `report_id` | FK → `reports.id` (cascade delete)  | Bejelentés azonosítója             |
| `user_id`   | FK → `users.id` (cascade delete)    | Szavazó felhasználó                |
| `vote_type` | enum: `up`, `down`                  | Szavazat típusa                   |
| `created_at`| timestamp, nullable                 | Szavazás ideje                    |
| `deleted_at`| timestamp, nullable                 | Soft delete ideje                 |

> **Egyedi megkötés:** `(report_id, user_id)` – egy felhasználó egy bejelentésre csak egyszer szavazhat.

---

## 4. Modellek és kapcsolatok

### `User`

- **Trait-ek:** `HasApiTokens`, `HasFactory`, `Notifiable`, `SoftDeletes`
- **Kapcsolatok:**
  - `reports()` → `hasMany(Report::class)`
  - `votes()` → `hasMany(Vote::class)`
- **Metódus:** `isAdmin(): bool` – visszaadja, hogy a `role` értéke `admin`-e

### `Report`

- **Trait-ek:** `HasFactory`, `SoftDeletes`
- **Kapcsolatok:**
  - `user()` → `belongsTo(User::class)`
  - `category()` → `belongsTo(Category::class)`
  - `images()` → `hasMany(ReportImage::class)`
  - `votes()` → `hasMany(Vote::class)`
- **Accessor:** `getCredibilityScoreAttribute(): int` – upvote−downvote különbsége

### `Category`

- **Trait-ek:** `HasFactory`, `SoftDeletes`
- **Kapcsolat:** `reports()` → `hasMany(Report::class)`

### `ReportImage`

- **Kapcsolat:** `report()` → `belongsTo(Report::class)`

### `Vote`

- **Trait:** `SoftDeletes`
- **Kapcsolatok:**
  - `report()` → `belongsTo(Report::class)`
  - `user()` → `belongsTo(User::class)`

---

## 5. Jogosultságkezelés

A rendszerben **három hozzáférési szint** létezik:

### Vendég (nem bejelentkezett)

- Jóváhagyott bejelentések listázása és megtekintése
- Térkép adatok lekérése
- Kategóriák listázása
- Bejelentés hitelességi pontszámának megtekintése
- Képek megtekintése
- Publikus statisztikák

### Felhasználó (`role = 'user'`)

Minden, amit a vendég elér, plusz:
- Saját profil megtekintése és szerkesztése
- Saját bejelentések listázása
- Bejelentés létrehozása (státusz automatikusan `pending` lesz)
- Saját bejelentés szerkesztése és törlése (soft delete)
- Képek feltöltése és törlése saját bejelentéshez
- Szavazás bejelentésekre (upvote/downvote, visszavonható)

### Admin (`role = 'admin'`)

Minden, amit a felhasználó elér, plusz:
- Összes bejelentés listázása státusztól függetlenül
- Bármely bejelentés törlése
- Bejelentés jóváhagyása / elutasítása
- Felhasználók listázása és keresése
- Felhasználó tiltása / tiltás feloldása (admin nem tiltható)
- Kategóriák létrehozása, szerkesztése, törlése
- Admin statisztikák (részletes, minden státuszra)

> **Megjegyzés:** Tiltott (`is_banned = true`) felhasználó bejelentkezéskor hibaüzenetet kap, tokennel érkező kéréseinél pedig `403` státuszkódot.

---

## 6. Middleware-ek

### `AdminMiddleware` (alias: `admin`)

Ellenőrzi, hogy a bejelentkezett felhasználónak admin szerepköre van-e. Ha nem → `403`.

### `CheckBanned` (alias: `check.banned`)

Ellenőrzi, hogy a bejelentkezett felhasználó nincs-e tiltva. Ha tiltva → `403`.

**Middleware lánc a védett végpontokon:**

```
auth:sanctum  →  check.banned  →  [admin (csak admin útvonalon)]
```

---

## 7. Form Request validáció

### `RegisterRequest`

| Mező       | Szabályok                                                     |
|------------|---------------------------------------------------------------|
| `name`     | kötelező, string, max 255 karakter                            |
| `email`    | kötelező, érvényes email, egyedi a `users` táblában           |
| `password` | kötelező, min 8 karakter, `confirmed` (megegyező megerősítés) |

### `StoreReportRequest`

| Mező          | Szabályok                                               |
|---------------|---------------------------------------------------------|
| `category_id` | kötelező, létező `categories.id`                        |
| `title`       | kötelező, string, max 255 karakter                      |
| `description` | kötelező, string                                        |
| `latitude`    | opcionális, szám, −90 és 90 között                      |
| `longitude`   | opcionális, szám, −180 és 180 között                    |
| `date`        | kötelező, érvényes dátum, nem lehet jövőbeli            |
| `witnesses`   | opcionális, egész szám, minimum 1                       |

> Üres string értékek `latitude`, `longitude`, `witnesses` mezőknél automatikusan `null`-ra konvertálódnak a `prepareForValidation` metódusban.

### `UpdateReportRequest`

Ugyanazok a szabályok, mint `StoreReportRequest`, de minden mező `sometimes` (nem kötelező). Frissítésnél csak a küldött mezők változnak.

### `VoteRequest`

| Mező        | Szabályok                          |
|-------------|------------------------------------|
| `vote_type` | kötelező, értéke: `up` vagy `down` |

### `StoreCategoryRequest` / `UpdateCategoryRequest`

| Mező          | Szabályok                             |
|---------------|---------------------------------------|
| `name`        | kötelező, string, max 255, egyedi     |
| `description` | opcionális, string                    |

---

## 8. API végpontok

### 8.1 Publikus végpontok (hitelesítés nélkül)

| Metódus | Végpont                              | Leírás                                               |
|---------|--------------------------------------|------------------------------------------------------|
| `POST`  | `/api/register`                      | Regisztráció, token visszaadása (`201`)               |
| `POST`  | `/api/login`                         | Bejelentkezés, token visszaadása (`200`)              |
| `GET`   | `/api/reports`                       | Jóváhagyott bejelentések listája (szűrhető, rendezhető) |
| `GET`   | `/api/reports/{report}`              | Bejelentés részletei                                 |
| `GET`   | `/api/map/reports`                   | Térkép nézethez koordináták                          |
| `GET`   | `/api/reports/{report}/credibility`  | Hitelességi pontszám (upvote−downvote)               |
| `GET`   | `/api/reports/{report}/images`       | Bejelentés képeinek listája                          |
| `GET`   | `/api/categories`                    | Kategóriák listája                                   |
| `GET`   | `/api/categories/{category}`         | Egy kategória részletei                              |
| `GET`   | `/api/statistics`                    | Publikus statisztikák                                |

**Szűrők és rendezés a `GET /api/reports` végponton:**

| Paraméter     | Leírás                                                       |
|---------------|--------------------------------------------------------------|
| `category_id` | Szűrés kategória szerint                                     |
| `date_from`   | Szűrés esemény dátumra (tól)                                 |
| `date_to`     | Szűrés esemény dátumra (ig)                                  |
| `sort_by`     | Rendezési alap: `created_at`, `date`, `title`, `credibility` |
| `sort_dir`    | Rendezési irány: `asc` vagy `desc`                           |

### 8.2 Védett végpontok (`auth:sanctum` + `check.banned`)

| Metódus  | Végpont                           | Leírás                                              |
|----------|-----------------------------------|-----------------------------------------------------|
| `POST`   | `/api/logout`                     | Kijelentkezés, token törlése                        |
| `GET`    | `/api/user`                       | Bejelentkezett felhasználó adatai                   |
| `GET`    | `/api/profile`                    | Saját profil (bejelentések száma is)                |
| `PUT`    | `/api/profile`                    | Profil szerkesztése (name, email, password)         |
| `GET`    | `/api/users/{userId}/reports`     | Felhasználó bejelentéseinek listája                 |
| `POST`   | `/api/reports`                    | Új bejelentés létrehozása (státusz: `pending`)      |
| `PUT`    | `/api/reports/{report}`           | Bejelentés szerkesztése (saját vagy admin)          |
| `DELETE` | `/api/reports/{report}`           | Bejelentés törlése – soft delete (saját vagy admin) |
| `POST`   | `/api/reports/{report}/images`    | Képek feltöltése (max 10 db, max 5 MB/kép)          |
| `DELETE` | `/api/images/{image}`             | Kép törlése (fizikai + soft delete)                 |
| `POST`   | `/api/reports/{report}/vote`      | Szavazás / visszavonás / módosítás                  |

### 8.3 Admin végpontok (`auth:sanctum` + `check.banned` + `admin`, prefix: `/api/admin`)

| Metódus  | Végpont                                | Leírás                                               |
|----------|----------------------------------------|------------------------------------------------------|
| `GET`    | `/api/admin/reports`                   | Összes bejelentés listája (szűrhető status szerint)  |
| `DELETE` | `/api/admin/reports/{report}`          | Bejelentés törlése                                   |
| `PUT`    | `/api/admin/reports/{report}/approve`  | Bejelentés jóváhagyása                               |
| `PUT`    | `/api/admin/reports/{report}/reject`   | Bejelentés elutasítása                               |
| `GET`    | `/api/admin/users`                     | Felhasználók listája (kereshető name és email alapján) |
| `PUT`    | `/api/admin/users/{user}/ban`          | Felhasználó tiltása (admin nem tiltható)             |
| `PUT`    | `/api/admin/users/{user}/unban`        | Tiltás feloldása                                     |
| `POST`   | `/api/admin/categories`                | Új kategória létrehozása                             |
| `PUT`    | `/api/admin/categories/{category}`     | Kategória szerkesztése                               |
| `DELETE` | `/api/admin/categories/{category}`     | Kategória törlése (soft delete)                      |
| `GET`    | `/api/admin/statistics`                | Részletes admin statisztikák                         |

---

## 9. Controllerek

### `AuthController`

| Metódus    | Leírás                                                                              |
|------------|--------------------------------------------------------------------------------------|
| `register` | `RegisterRequest` alapú validáció, `role = 'user'`, Sanctum token generálás, `201` |
| `login`    | Email/jelszó ellenőrzés, tiltás vizsgálat (`403`), token generálás                 |
| `logout`   | Aktuális access token törlése                                                       |
| `user`     | Bejelentkezett felhasználó adatainak visszaadása                                    |

---

### `ReportController`

| Metódus      | Hozzáférés | Leírás                                                                   |
|--------------|------------|--------------------------------------------------------------------------|
| `index`      | Publikus   | Csak `approved` bejelentések, szűrők és rendezés támogatásával           |
| `show`       | Vegyes     | `approved` → mindenki; más státusz → csak tulajdonos vagy admin          |
| `store`      | Védett     | Státusz automatikusan `pending`, `user_id` az autentikált felhasználóból |
| `update`     | Védett     | Csak tulajdonos vagy admin (`403` egyébként)                             |
| `destroy`    | Védett     | Soft delete, csak tulajdonos vagy admin (`403` egyébként)                |
| `mapReports` | Publikus   | Koordinátával rendelkező, jóváhagyott bejelentések minimális adatai      |

---

### `VoteController`

**`vote`** – három eset kezelése:

| Eset                       | Eredmény                    |
|----------------------------|-----------------------------|
| Ugyanolyan szavazat újra   | `forceDelete` – visszavonás |
| Ellentétes szavazat        | `update` – módosítás        |
| Nincs még szavazat         | `create` – új szavazat      |

> `forceDelete` azért szükséges, mert a `UNIQUE(report_id, user_id)` DB-megkötés a soft-deleted rekordokra is érvényes.

Visszaadja az aktuális `upvotes`, `downvotes` és `credibility_score` értékeket.

**`credibility`** – publikus, upvote/downvote arány és pontszám.

---

### `ImageController`

| Metódus   | Hozzáférés | Leírás                                                                                                    |
|-----------|------------|-----------------------------------------------------------------------------------------------------------|
| `index`   | Publikus   | Bejelentés képeinek listázása                                                                             |
| `store`   | Védett     | Csak tulajdonos vagy admin; max 10 kép; max 5 MB; formátum: jpeg/png/jpg/gif/webp; hash-elt fájlnévvel `public/storage/report_images/`-be kerül |
| `destroy` | Védett     | Csak tulajdonos vagy admin; fizikai fájl törlés + soft delete a rekordból                                 |

---

### `ProfileController`

| Metódus       | Leírás                                                              |
|---------------|---------------------------------------------------------------------|
| `show`        | Saját felhasználói adatok + `reports_count`                         |
| `update`      | `name`, `email`, `password` módosítható; jelszó bcrypt-tel tárolva  |
| `userReports` | Adott `userId` összes bejelentése szavazatszámokkal és kategóriával |

---

### `CategoryController`

| Metódus   | Hozzáférés | Leírás                   |
|-----------|------------|--------------------------|
| `index`   | Publikus   | Kategóriák listázása     |
| `show`    | Publikus   | Egy kategória részletei  |
| `store`   | Admin only | Új kategória létrehozása |
| `update`  | Admin only | Kategória szerkesztése   |
| `destroy` | Admin only | Soft delete              |

---

### `StatisticsController` (publikus)

Visszaadja:
- Jóváhagyott bejelentések száma
- Összes felhasználó és szavazat száma
- Státusz szerinti bontás (`pending`, `approved`, `rejected`)
- Kategóriánkénti bontás (csak jóváhagyott)
- Top 5 leghitelesebb bejelentés (hitelesség-pontszám alapján)
- Legutóbbi 5 jóváhagyott bejelentés

---

### `Admin\ReportController`

| Metódus   | Leírás                                            |
|-----------|---------------------------------------------------|
| `index`   | Összes bejelentés, opcionális `?status=` szűrővel |
| `destroy` | Soft delete                                       |
| `approve` | Státusz → `approved`                              |
| `reject`  | Státusz → `rejected`                              |

---

### `Admin\UserController`

| Metódus | Leírás                                                                 |
|---------|------------------------------------------------------------------------|
| `index` | Felhasználók listája `reports_count`-tal; kereshető `?search=` alapján |
| `ban`   | Admin nem tiltható (`422`); user esetén `is_banned = true`             |
| `unban` | `is_banned = false`                                                    |

---

### `Admin\StatisticsController`

Visszaadja:
- Bejelentések száma státusz szerint (total, pending, approved, rejected)
- Felhasználók száma (total, banned)
- Összes szavazat és kategória száma
- Top 5 legmagasabb hitelességű bejelentés
- Kategóriánkénti bejelentésszám

---

## 10. Seederek

A `DatabaseSeeder` az alábbi seedereket futtatja sorban:

| Seeder           | Leírás                                                   |
|------------------|----------------------------------------------------------|
| `CategorySeeder` | 9 előre definiált kategóriát hoz létre (`firstOrCreate`) |
| `UserSeeder`     | Teszt felhasználókat hoz létre (admin és sima user)      |
| `ReportSeeder`   | Mintabejelentéseket hoz létre                            |
| `VoteSeeder`     | Mintaszavazatokat hoz létre                              |

**Előre feltöltött kategóriák:**

| # | Kategória           | Leírás                                          |
|---|---------------------|-------------------------------------------------|
| 1 | UFO Észlelés        | Azonosítatlan repülő tárgyak és légi jelenségek |
| 2 | Földönkívüli        | Idegen lények találkozásai                      |
| 3 | Kísértet / Szellem  | Természetfeletti entitások, kísértett helyek    |
| 4 | Crop Circle         | Rejtélyes búzamező alakzatok, leszállási nyomok |
| 5 | Bigfoot / Sasquatch | Nagy emberszabású lény észlelések               |
| 6 | Tengeri Szörny      | Azonosítatlan vízi lények                       |
| 7 | Poltergeist         | Zajokkal, tárgyak mozgásával járó jelenségek    |
| 8 | Időhurok / Anomália | Idővel kapcsolatos furcsa tapasztalatok         |
| 9 | Egyéb Paranormális  | Minden más megmagyarázhatatlan jelenség         |

**Seederek futtatása:**

```bash
php artisan db:seed
```

---

## 11. Tesztek

### PHPUnit Feature tesztek

A tesztek a `tests/Feature/` könyvtárban találhatók, futtatásuk:

```bash
php artisan test
```

### Postman kollekció

A `ufo-api.postman_collection.json` fájl importálható közvetlenül Postmanbe.

**Import lépések:**
1. Postman → **Import** gomb
2. Fájl kiválasztása: `backend/ufo-api.postman_collection.json`
3. Environment létrehozása: `base_url = http://localhost:8000/api`

**Tipikus tesztelési folyamat Postmanben:**

```
1. POST /api/register                    →  token kimentése
2. POST /api/login                       →  token kimentése
3. POST /api/reports                     →  Bearer token fejléccel
4. POST /api/reports/{id}/vote           →  szavazás tesztelése
5. PUT  /api/admin/reports/{id}/approve  →  admin tokennel
```

#### 1. Regisztráció / Bejelentkezés
POST /api/register

![Regisztráció – Postman](backend/images/register.png)

A álaszban kapott `token` értékét az `Authorization: Bearer <token>` fejlécbe kell másolni.

#### 2. Bejelentés létrehozása
POST /api/reports

![Bejelentkezés – Postman](backend/images/userlogin.png)

#### 3. Szavazás
POST /api/reports/2/vote

![Felszavazás – Postman](backend/images/upvote.png)

#### 4. Admin: Bejelentés jóváhagyása
PUT /api/admin/reports/1/approve

![Jóváhagyás – Postman](backend/images/adminapprove.png)
#### 5. Az össze bejelentés megnézése
GET /api/reports

![Jóváhagyás – Postman](backend/images/allreports.png)

### HTTP válaszkódok

| Kód | Jelentés                                     |
|-----|----------------------------------------------|
| 200 | Sikeres lekérdezés / művelet                 |
| 201 | Sikeres létrehozás                           |
| 401 | Nem hitelesített (hiányzó/érvénytelen token) |
| 403 | Hozzáférés megtagadva (nem admin / tiltva)   |
| 404 | Az erőforrás nem található                   |
| 422 | Validációs hiba (hibás bemeneti adatok)      |
| 500 | Szerver oldali hiba                          |

---

# FRONTEND

---

## 12. Frontend áttekintés

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

## 13. Technológiai stack (frontend)

| Komponens       | Verzió / Leírás                                    |
|-----------------|----------------------------------------------------|
| Angular         | 19 (standalone components, signals)                |
| TypeScript      | 5.x                                                |
| Bootstrap       | 5.3.3 (CDN, scss változók felülírva)               |
| Bootstrap Icons | 1.11.3 (CDN)                                       |
| Leaflet.js      | Térkép megjelenítés                                |
| RxJS            | HTTP kommunikáció, debounce szűrés                 |
| Angular Signals | Reaktív állapotkezelés (`AuthService.currentUser`) |
| Google Fonts    | Orbitron, Share Tech Mono, Inter                   |

---

## 14. Projektstruktúra

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

## 15. Környezeti konfiguráció

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

## 16. Útvonalak és oldalak

Az összes útvonal lazy-loaded – az oldal kódja csak az első látogatáskor töltődik le.

| Útvonal             | Komponens         | Guard        | Leírás                                        |
|---------------------|-------------------|--------------|-----------------------------------------------|
| `/`                 | `ReportList`      | —            | Bejelentések listája, slider, szűrés, lapozás |
| `/reports/:id`      | `ReportDetail`    | —            | Bejelentés részletei, térkép, szavazás, képek |
| `/reports/create`   | `ReportForm`      | `authGuard`  | Új bejelentés létrehozása                     |
| `/reports/:id/edit` | `ReportForm`      | `authGuard`  | Meglévő bejelentés szerkesztése               |
| `/statistics`       | `Statistics`      | —            | Statisztikák, top lista, kategória bontás     |
| `/profile`          | `Profile`         | `authGuard`  | Saját adatok, jelszó, saját bejelentések      |
| `/login`            | `Login`           | —            | Bejelentkezési űrlap                          |
| `/register`         | `Register`        | —            | Regisztrációs űrlap                           |
| `/admin/reports`    | `AdminReports`    | `adminGuard` | Státuszszűrés, jóváhagyás, elutasítás, törlés |
| `/admin/users`      | `AdminUsers`      | `adminGuard` | Felhasználók listázása, keresés, tiltás       |
| `/admin/categories` | `AdminCategories` | `adminGuard` | Kategória hozzáadás, szerkesztés, törlés      |
| `**`                | —                 | —            | Ismeretlen útvonal → `/`                      |

---

## 17. Komponensek

### ReportList (főoldal)

<img width="1920" height="1080" alt="Képernyőfelvétel (629)" src="https://github.com/user-attachments/assets/f6c6feca-49b6-426a-9b65-efa9dffe5882" />

A főoldalon három fő rész található:

**1. Slider** – A top 3 leghitelesebb bejelentés auto-lejátszós csúszkán jelenik meg (5 másodpercenként vált, manuálisan is léptethető). A hitelességi pontszám = upvote − downvote.

**2. Szűrő és rendező panel** – Kategória, dátum intervallum és rendezési szempont választható.

| Rendezési lehetőség             | Paraméter                        |
|---------------------------------|----------------------------------|
| Feltöltés dátuma (újabb elöl)   | `created_at` / `desc` (alapért.) |
| Feltöltés dátuma (régebbi elöl) | `created_at` / `asc`             |
| Esemény dátuma (újabb elöl)     | `date` / `desc`                  |
| Esemény dátuma (régebbi elöl)   | `date` / `asc`                   |
| Cím (A → Z)                     | `title` / `asc`                  |
| Cím (Z → A)                     | `title` / `desc`                 |
| Hitelesség (nagyobb elöl)       | `credibility` / `desc`           |
| Hitelesség (kisebb elöl)        | `credibility` / `asc`            |

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

| Mező           | Típus          | Kötelező | Megjegyzés                            |
|----------------|----------------|----------|---------------------------------------|
| Kategória      | select         | igen     | API-ból töltődik be                   |
| Cím            | text           | igen     | max 255 karakter                      |
| Leírás         | textarea       | igen     | –                                     |
| Esemény dátuma | date           | igen     | Nem lehet jövőbeli                    |
| Tanúk száma    | number         | nem      | Egész szám, minimum 1                 |
| Helyszín       | Leaflet térkép | nem      | Kattintásra beállítja a koordinátákat |
| Képek          | file input     | nem      | Max 10 db, max 5 MB/kép, előnézet     |

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
- Szerkeszthető: név, e-mail, jelszó (jelszó csak akkor kerül el, ha ki van töltve)
- Alul listázza a felhasználó összes saját bejelentését státuszjelzőkkel és linkekkel

---

### Login / Register

<img width="1920" height="1080" alt="Képernyőfelvétel (637)" src="https://github.com/user-attachments/assets/f4d90220-cb9c-4ee6-b8d7-01680a47a223" />
<img width="1920" height="1080" alt="Képernyőfelvétel (638)" src="https://github.com/user-attachments/assets/99059bed-5236-43f1-9318-024421a2d03b" />

Egyszerű űrlapok Template-driven FormsModule-lal. A backend validációs hibákat mezőnként kapja meg az alkalmazás, összefűzve jeleníti meg egy hibaüzenetben. Sikeres bejelentkezés / regisztráció után a főoldalra navigál.

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

| Bejelentkezve | Admin | Megjelenő elemek                                       |
|---------------|-------|--------------------------------------------------------|
| nem           | nem   | Bejelentések, Statisztikák, Bejelentkezés, Regisztráció |
| igen          | nem   | + Új bejelentés, Profil, Kijelentkezés                 |
| igen          | igen  | + Admin menü (Bejelentések, Felhasználók, Kategóriák)  |

---

### Map (komponens)

Leaflet.js alapú térkép, két módban:

| Mód              | Input            | Output       | Leírás                                          |
|------------------|------------------|--------------|-------------------------------------------------|
| Csak megjelenítő | `[lat]`, `[lng]` | —            | Koordináta alapján mutatja a helyszínt, readonly |
| Interaktív       | `[clickable]=true` | `(mapClick)` | Kattintáskor visszaadja `[lat, lng]` tömböt   |

A `height` input (pl. `"300px"`) dinamikusan állítja a doboz magasságát.

```html
<!-- Megjelenítő mód -->
<app-map [lat]="report.latitude" [lng]="report.longitude" [clickable]="false" height="300px" />

<!-- Interaktív mód -->
<app-map [clickable]="true" (mapClick)="onMapClick($event)" height="400px" />
```

A Leaflet OpenStreetMap csempéket használ. A `invalidateSize()` meghívása fontos, különben szürke csempék jelennek meg, ha a konténer mérete az inicializálás után változott.

---

## 18. Szolgáltatások

### AuthService

A bejelentkezett felhasználó állapotát Angular **Signal**-ban (`currentUser`) tárolja. Az adat a `sessionStorage`-ban marad, így oldal-újratöltés után sem vész el a munkamenet.

| Metódus        | Leírás                                                          |
|----------------|-----------------------------------------------------------------|
| `register()`   | POST `/api/register` → token és user mentése sessionStorage-ba |
| `login()`      | POST `/api/login` → token és user mentése sessionStorage-ba    |
| `logout()`     | POST `/api/logout`, sessionStorage törlése, navigál `/login`-ra |
| `getToken()`   | Visszaadja a tárolt Bearer tokent                               |
| `isLoggedIn()` | `true` ha van token                                             |
| `isAdmin()`    | `true` ha `currentUser().role === 'admin'`                      |

---

### ReportService

| Metódus           | Végpont                        | Leírás                       |
|-------------------|--------------------------------|------------------------------|
| `getAll(filters)` | GET `/api/reports`             | Szűrők query paraméterként   |
| `getOne(id)`      | GET `/api/reports/:id`         | –                            |
| `create(data)`    | POST `/api/reports`            | –                            |
| `update(id,data)` | PUT `/api/reports/:id`         | –                            |
| `delete(id)`      | DELETE `/api/reports/:id`      | Soft delete                  |
| `getMapReports()` | GET `/api/map/reports`         | Koordinátás bejelentések     |
| `uploadImages()`  | POST `/api/reports/:id/images` | `FormData` – `images[]` tömb |

---

### VoteService

Egyetlen metódus: `vote(reportId, 'up' | 'down')` → POST `/api/reports/:id/vote`

Ugyanolyan szavazat = visszavonás. Ellentétes szavazat = felülírás. A válasz tartalmazza az aktuális upvote/downvote számlálókat.

---

### CategoryService

| Metódus        | Leírás                             |
|----------------|------------------------------------|
| `getAll()`     | GET `/api/categories`              |
| `create(data)` | POST `/api/admin/categories`       |
| `update(id)`   | PUT `/api/admin/categories/:id`    |
| `delete(id)`   | DELETE `/api/admin/categories/:id` |

---

### AdminService

| Metódus              | Végpont                              | Leírás                  |
|----------------------|--------------------------------------|-------------------------|
| `getReports(status)` | GET `/api/admin/reports?status=`     | Opcionális státuszszűrő |
| `approveReport(id)`  | PUT `/api/admin/reports/:id/approve` | –                       |
| `rejectReport(id)`   | PUT `/api/admin/reports/:id/reject`  | –                       |
| `deleteReport(id)`   | DELETE `/api/admin/reports/:id`      | –                       |
| `getUsers(search)`   | GET `/api/admin/users?search=`       | Opcionális keresőszó    |
| `banUser(id)`        | PUT `/api/admin/users/:id/ban`       | –                       |
| `unbanUser(id)`      | PUT `/api/admin/users/:id/unban`     | –                       |

---

## 19. Guard-ok és interceptor

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

## 20. Stílus rendszer

A globális stílus (`src/styles.css`) egy sötét sci-fi témát valósít meg Bootstrap változók felülírásával.

### CSS változók (`:root`)

| Változó         | Érték     | Felhasználás                       |
|-----------------|-----------|------------------------------------|
| `--sf-bg`       | `#0b0e14` | Oldal háttere                      |
| `--sf-surface`  | `#111620` | Kártyák, táblázatok háttere        |
| `--sf-surface2` | `#161c2a` | Form elemek, kártya fejléc háttere |
| `--sf-border`   | `#1e2d45` | Körvonalak, elválasztók            |
| `--sf-accent`   | `#3a8fc8` | Kék kiemelőszín (gombok, linkek)   |
| `--sf-accent2`  | `#4db8c8` | Élénkebb kék (hover, brand)        |
| `--sf-text`     | `#dde6f5` | Fő szövegszín                      |
| `--sf-text-dim` | `#8a9dba` | Halvány szöveg, labels             |

### Betűtípusok

- **Orbitron** – brand logó, statisztika számok (sci-fi megjelenés)
- **Share Tech Mono** – badge-ek, kategóriacímkék
- **Inter** – általános szöveg

### Bootstrap felülírás

A Bootstrap saját CSS változói (`--bs-body-bg`, `--bs-card-bg`, stb.) a `:root`-ban kerülnek felülírásra, így az összes Bootstrap komponens automatikusan a sci-fi témát használja, külön osztályok nélkül.


