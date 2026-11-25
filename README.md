# Mon Constat Digital

[![CI](https://github.com/Salma-Louhichi/Amio/actions/workflows/ci.yml/badge.svg)](https://github.com/Salma-Louhichi/Amio/actions/workflows/ci.yml)
![Flutter](https://img.shields.io/badge/Flutter-3-02569B?logo=flutter)
![Node.js](https://img.shields.io/badge/Node.js-22-339933?logo=node.js)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql)

**Mon Constat Digital** is a digital accident report (*constat amiable*): two drivers fill in, sign and seal their report on a phone, then follow the claim through to settlement.

A Flutter app (Android, iOS, web) in French, English and Arabic, backed by a Node.js / PostgreSQL API. Integrated project (PI), 5SAE1, Esprit.

## Features

| | |
|---|---|
| **Accounts and identity** | Two-step sign-up, roles (driver, agent, administrator), biometric sign-in, AI identity verification (face + ID document, MRZ, driving licence). |
| **Vehicles and contracts** | Vehicles, insurance contracts, reminders before documents expire. |
| **Accident declaration** | Step-by-step wizard with photos, GPS, sketch and an FTUSA-compliant PDF. |
| **Digital report** | Driver B invited by QR code, the 17 official circumstances, dual signature, ECDSA P-256 sealing and a signed PDF. |
| **Public verification** | [`verifier-pwa/`](verifier-pwa/): a zero-dependency PWA that checks a sealed report (QR code or PDF), offline. |
| **Claims** | Real-time tracking, agent / driver messaging, expert appointments, AI summaries and assistant (Claude, OpenAI or local rules), analytics dashboard. |
| **Privacy** | Explicit, dated consent (Tunisian law no. 2004-63), personal data export as PDF, account deletion. |

## Architecture

```
Flutter app (drift, offline-first) ──HTTPS/JWT──▶ Express API ──Prisma──▶ PostgreSQL
        │                                         │
        └── Signed PDF + QR (JWS) ──▶ verifier-pwa ├── Socket.io (real time)
                                                  └── AI: Claude / OpenAI, OCR, face
```

## Quality

- `flutter analyze` reports no issues; 230+ Flutter tests.
- 220+ API tests against a real PostgreSQL database.
- Both suites run in GitHub Actions on every push ([`ci.yml`](.github/workflows/ci.yml)).

## Getting started

### Flutter app

Requirements: Flutter stable (Dart 3.10 or later). For Android, also install Android Studio.

> **Windows:** clone the project into a path without accents or spaces, for example `C:\dev\mon-constat`. With an accent in the path, `flutter analyze` crashes and Android builds can fail. Also enable Developer Mode (`start ms-settings:developers`), which Flutter plugins need.

```bash
flutter pub get
flutter devices          # list available devices
flutter run -d chrome    # quickest way to check a screen
flutter run              # on the connected phone or emulator
```

The app needs the API to sign in (see below). On a connected phone or emulator, forward the API port first:

```bash
adb reverse tcp:3000 tcp:3000
```

To target another server: `flutter run --dart-define=API_URL=http://192.168.1.10:3000/api`.

> **Java/Gradle error on the first Android build:** recent Android Studio ships Java 25, too new for the project's Gradle version. Install JDK 17, then: `flutter config --jdk-dir="C:\Program Files\Java\jdk-17"`.
>
> **Huawei / Xiaomi:** on the first `flutter run`, accept the "Install via USB" prompt on the phone.

Before each push:

```bash
dart format lib test
flutter analyze
flutter test
```

### Admin console (web)

The same app serves as the administrators' console on desktop: from 1000 px wide, an administrator account gets a sidebar (dashboard, accounts, identity verification with the AI score, companies) instead of the phone tabs.

```bash
flutter run -d chrome                     # development
flutter build web                         # deployable build (build/web folder)
```

The SQLite databases (drift) also work in the browser thanks to `web/sqlite3.wasm` and `web/drift_worker.js` (drift release 2.35.3); update them along with the drift package. Every database passes `web: driftWebOptions` (`lib/core/data/drift_web.dart`).

### API and database

Requirement: Docker.

```bash
docker compose up --build    # API on http://localhost:3000, PostgreSQL on port 5434
```

Check: `http://localhost:3000/api/health` returns `{"status":"ok"}`.

Demo accounts created at startup (password `Password123!`):

| Role | Email |
|---|---|
| Driver | `sami@example.com` |
| Agent | `leila@assurance.tn` |
| Administrator | `admin@monconstat.tn` |

With no provider configured, OTP and password-reset codes are printed in the logs: `docker compose logs api`.

To develop the API outside a container:

```bash
docker compose up db
cd backend
cp .env.example .env
npm install
npm run db:deploy            # apply migrations
npm run db:seed              # demo accounts
npm run dev
npm test                     # needs the database: docker compose up db
```

To change the schema: edit `backend/prisma/schema.prisma`, then run `npm run db:migrate`.

API contract (Swagger): `http://localhost:3000/api/docs`, generated from `backend/openapi.yaml`. Each module documents its endpoints there; a test checks that every documented route exists.

## Code layout

```text
lib/
  app/                 Startup, role-based navigation, home, support
  core/
    theme/             Colours and theme (DESIGN.md)
    state/             AppScope: access to the stores
  shared/              Widgets shared across modules
  modules/
    users/             Module 1
    vehicles/          Module 2
    accident/          Module 3
    constat/           Module 4
    claims/            Module 5
backend/src/modules/   The same modules on the API side
```

Each Flutter module follows the same structure: `models/`, `state/`, `screens/`, `widgets/`. Each backend module adds its endpoints in its own `<module>.routes.js` file.

| Module | Scope | Flutter | Backend | Depends on |
|---|---|---|---|---|
| 1 | Users, roles, administration | `modules/users` | `users` | None |
| 2 | Vehicles and contracts | `modules/vehicles` | `vehicles` | 1 |
| 3 | Accident declaration | `modules/accident` | `accidents` | 1, 2 |
| 4 | Digital report | `modules/constat` | `constats` | 3 |
| 5 | Claims, tracking, communication | `modules/claims` | `claims` | 4 |

### Using authentication in a module

On the API side, protect routes with the module 1 middleware:

```js
import { requireAuth, requireRole } from '../../middleware/auth.js';

router.use(requireAuth);                              // req.user holds the signed-in account
router.get('/', requireRole('AGENT', 'ADMIN'), ...);  // restrict to some roles
```

On the Flutter side, go through the shared client. It adds the token, refreshes it when needed and throws an `ApiException` whose message can be shown to the user:

```dart
final response = await AppScope.of(context).api.get('/vehicles');
final user = AppScope.of(context).users.currentUser;
// Companies and agencies (module 1), available offline through SQLite:
await AppScope.of(context).companies.load();
final companies = AppScope.of(context).companies.companies;
final star = AppScope.of(context).companies.byId(companyId);
```

### Insurance company: from contract to agent

For a claim to reach the agents of the right company, every link in the chain keeps the company's **ID** (not its name as free text):

```text
Contract (module 2) --companyId--> Accident (module 3, declared vehicle) --> Claim (module 5, companyId) --> that company's agents
```

- **Module 2:** pick the insurer with `CompanyPicker` (the module 1 list, available offline) and store `company.id` (and `agency.id`) on the contract. On the API side, check it with `assertCompanyExists(companyId)`.
- **Module 3:** store the vehicle involved in the declaration.
- **Module 5:** copy the contract's `companyId` onto the claim, and filter lists with `companyScope(req)`: an agent only sees their own company, an administrator sees everything.

```dart
CompanyPicker(
  companyId: contract.companyId,
  onChanged: (company, agency) => setState(() => contract.companyId = company?.id),
)
```

```js
import { assertCompanyExists, companyScope, requireAuth } from '../../middleware/auth.js';

await assertCompanyExists(req.body.companyId);                       // module 2
prisma.claim.findMany({ where: { ...companyScope(req), status } });   // module 5
```

### AI identity verification

The driver picks their document (national ID card, driving licence or passport), photographs it, then completes a **face check**: the API randomises the order of the prompts (face forward, then head turned left and right) and the app takes one photo per prompt.

Face recognition runs **inside the API**, with no external service ([`backend/src/lib/face.js`](backend/src/lib/face.js)): face-api and TensorFlow.js on WebAssembly. Three models: face detection (SSD MobileNet v1), 68 facial landmarks, and a 128-number face descriptor (ResNet-34).

1. **Front of the document:** rejected if it contains no face. A rotated photo is turned upright (all 4 orientations are tried). The AI reads the document ([`backend/src/lib/ocr.js`](backend/src/lib/ocr.js), Tesseract): the number and Arabic fields of the ID card, the passport MRZ with its check digits, the licence number.
2. **Back:** rejected if it shows a face (that's a front side). A recognised barcode or issue date counts towards automatic approval.
3. **Liveness:** on the phone, ML Kit detects the face and takes each photo at the right moment (a countdown on the web). On the server: one face per photo, facing forward then turned to both sides, and the same person in all three photos. A printed photo or a screen doesn't pass.
4. **Decision:** approved automatically only if the face matches the document photo (at least 60 %), the number entered is the one read, the back is recognised, the account name appears on the document (in Arabic, compared by consonants: الطرابلسي ≈ Trabelsi) and the document is valid. Otherwise it goes to the administrator's queue, with the list of checks.
5. **Rejection:** 24 h wait, at most 3 rejections; a serious rejection blocks the account. The administrator can unblock it and sees the history of requests.

Cross-checks: the date of birth on the ID document is compared with the one on the driving licence scanned in the profile. The dashboard shows the AI's track record (share approved automatically, average similarity, blocked accounts).

Under law no. 2004-63, face analysis requires **explicit, dated consent**, no photo is sent to a third party, the face descriptor is never stored and the face photo is **deleted as soon as a decision is made**. Each account gets 30 analyses per hour, so no one can overload the server. Other modules require a verified identity through `requireVerifiedIdentity`.

## Git workflow

1. Create a branch from an up-to-date `main`: `git switch -c feature/<topic>`.
2. Keep changes inside the folder of the module concerned.
3. Open a pull request to `main`; CI must pass before merging.

**Rules:**

- Don't push directly to `main`.
- The Prisma schema and the API contract (Swagger) are updated before the code.
- No Firebase.
- Never commit secrets: the `.env` file stays local.

## Design

The whole interface follows [DESIGN.md](DESIGN.md).

- Colours: `context.palette.surface`, `context.palette.text`… (`lib/core/theme/app_palette.dart`), never a hard-coded colour or `Colors.white` / `Colors.grey`, or the screen breaks in dark mode. `AppColors` is kept for what never changes (amber, drivers A and B).
- Strings: in `lib/l10n/app_fr.arb` (then `app_en.arb`, `app_ar.arb`), used through `context.t.myKey`. Arabic is right-to-left: prefer `EdgeInsetsDirectional` and `PositionedDirectional`.
- Fonts: Manrope, Bricolage Grotesque and JetBrains Mono, already bundled in `assets/fonts/`.
- Driver A is always blue and driver B always amber.
