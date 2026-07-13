# THINKGlobalSchool — System Architecture

## System Overview

THINKGlobalSchool's application ecosystem consists of two student-facing mobile apps (THINKStatus and THINKWallet), a shared Django backend (THINK-Wallet-backend), and a shared iOS signing certificate repository (THINKapps-certificates). The system manages student location status tracking and cash/expense management for students and staff at Think Global School.

```
┌─────────────────────┐     ┌─────────────────────┐
│  THINKStatus        │     │  THINKWallet         │
│  (React Native)     │     │  (React Native/Expo) │
│  Location tracking  │     │  Expense management  │
│  Medical symptoms   │     │  Cash transactions   │
└────────┬────────────┘     └────────┬─────────────┘
         │                           │
         │ POST /hubspot/status/     │ REST API (JWT)
         │                           │
         ▼                           ▼
┌────────────────────────────────────────────────┐
│           THINK-Wallet-backend                  │
│           Django 5.2 / DRF / PostgreSQL         │
│                                                  │
│  • User management (Google/Firebase auth)        │
│  • Ledger & transactions                         │
│  • File storage (AWS S3)                         │
│  • Email notifications (SendGrid)                │
│  • HubSpot CRM integration                       │
│  • Deployed on Heroku                            │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│           THINKapps-certificates                │
│           fastlane match (iOS signing)          │
│                                                  │
│  • Development & distribution certs              │
│  • Provisioning profiles for:                    │
│    - org.thinkglobalschool.status                │
│    - org.thinkglobalschool.wallet                │
└────────────────────────────────────────────────┘
```

---

## Repository Roles

### 1. THINKStatus-ReactNative

**Purpose:** Student location status and (legacy) medical symptom tracking app.

**Tech Stack:**
- React Native 0.74 (bare workflow, not Expo)
- TypeScript
- Redux Toolkit for state management
- React Navigation (drawer navigator)
- `simpleddp` — originally connected via Meteor DDP protocol (now replaced)
- `react-native-background-geolocation` for continuous location tracking
- Google Sign-In for authentication

**Entry Point:** `App.tsx` → `src/index.ts` → `src/status-app.tsx`

**Key Connections:**
- Sends location updates to `THINK-Wallet-backend` via `POST /hubspot/status/` (see `src/utils/utility-belt-client.ts`)
- Uses an API key (`API_KEY` env var) for the status endpoint
- Google Sign-In configured with iOS client ID `102734133751-...`
- Previously used DDP (Meteor) to connect to a "utility belt" server; now replaced by direct REST calls to the Wallet backend

**Configuration:**
- `.env` via `react-native-dotenv`: `API_KEY`, `BASE_URL`
- Default `BASE_URL`: `http://localhost:8000`

**Screens:** Login, Home (map with location), Settings, About, Med Symptoms (legacy/deprecated)

---

### 2. THINK-Wallet-backend

**Purpose:** Central Django REST API backend powering the THINKWallet app and providing the HubSpot status endpoint used by THINKStatus.

**Tech Stack:**
- Django 5.2 / Django REST Framework 3.16
- PostgreSQL (via `psycopg`)
- JWT authentication (`djangorestframework-simplejwt`)
- Firebase Admin SDK (for verifying Google ID tokens)
- AWS S3 / boto3 (file storage for receipts)
- SendGrid (email notifications)
- HubSpot API (CRM integration for status tracking)
- WhiteNoise (static files on Heroku)
- Gunicorn (WSGI server)
- Deployed on **Heroku** (primary) / also configured for Render

**Entry Point:** `manage.py` → `tgswallet/urls.py`

**API Endpoints:**
| Path | Module | Description |
|------|--------|-------------|
| `/users/` | `users/` | User management, Google auth (`/auth/google/`), token refresh (`/auth/refresh/`), `me/` |
| `/transactions/` | `ledger/urls2.py` | CRUD transactions, pending approvals, approve/reject |
| `/ledgers/` | `ledger/urls.py` | Ledger management, stats |
| `/categories/` | `ledger/urls3.py` | GL/Program category management |
| `/storage/` | `storage/` | File upload (S3 presigned URL flow) |
| `/config/` | `config/` | App configuration (default currency) |
| `/hubspot/` | `hubspot/` | HubSpot integration (status endpoint used by THINKStatus) |
| `/admin/` | Django admin | Admin UI (Jazzmin-themed) |
| `/swagger/` | drf-yasg | API docs (debug mode only) |

**User Roles:** Student, Advisor, Country Coordinator, Finance, Admin — with role-based permissions.

**Configuration (`.env`):**
- `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS`
- `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_HOST`, `POSTGRES_PORT` (or `DATABASE_URL` for Heroku)
- `FIREBASE_SERVICE_ACCOUNT` (JSON string)
- `S3_BUCKET_NAME`, `AWS_REGION`
- `SENDGRID_API_KEY`
- `HUBSPOT_ACCESS_TOKEN`, `STATUS_API_KEY`
- `BASE_URL`, `PLATFORM` (set to `heroku` for production)

---

### 3. THINK-Wallet-prod

**Purpose:** Student/staff expense management app — cash given, cash spent, cash returned, with receipt photo capture and approval workflows.

**Tech Stack:**
- React Native 0.79 via **Expo 53** (prebuilt workflow)
- TypeScript
- Redux Toolkit for state management
- React Navigation v7 (stack + bottom tabs)
- Firebase Auth (`@react-native-firebase/auth`)
- Google Sign-In (`@react-native-google-signin/google-signin`)
- React Hook Form + Yup (form validation)
- Expo Image Picker (receipt photos)
- Expo Secure Store (JWT token storage)
- Expo File System (S3 presigned URL uploads)

**Entry Point:** `index.ts` → `App.tsx` → `src/navigation/RootNavigator.tsx`

**Key Connections:**
- Authenticates via Firebase/Google → sends Firebase ID token to backend `/users/auth/google/` → receives JWT access + refresh tokens
- All subsequent API calls use JWT Bearer auth
- Uploads receipt images via S3 presigned URL flow (`/storage/upload/`)
- Default API base URL: `https://think-wallet-f490339730c6.herokuapp.com` (fallback: `https://tgswallet-backend.onrender.com`)

**Configuration:**
- `app.config.js` reads from `env.example`: `API_BASE_URL`, `GOOGLE_IOS_CLIENT_ID`, `APP_ENV`, `DEBUG_MODE`
- iOS bundle identifier: `org.thinkglobalschool.wallet`
- Google iOS Client ID: `450493643192-...`

**Screens:** Login, Dashboard, Transaction List/Detail/Create/Edit, Approval List

---

### 4. THINKapps-certificates

**Purpose:** Shared iOS code signing repository managed by [fastlane match](https://docs.fastlane.tools/actions/match/).

**Contents:**
- `certs/development/` — Development certificate (`.cer` + `.p12`) for team key `5LN3Z5YS86`
- `certs/distribution/` — Distribution certificate (`.cer` + `.p12`) for team key `GSK48XRF6G`
- `profiles/development/` — Development provisioning profiles for:
  - `org.thinkglobalschool.wallet`
  - `org.thinkglobalschool.status`
- `profiles/adhoc/` — Ad Hoc provisioning profiles for:
  - `org.thinkglobalschool.wallet`
  - `org.thinkglobalschool.status`

**Usage:** Both THINKStatus and THINKWallet iOS builds reference this repo for signing. Run `fastlane match development` (or `adhoc`/`appstore`) from either app project to pull certificates.

**Note:** This is NOT a duplicate — there is only one certificates repo. It serves both apps.

---

## Data Flow

### THINKStatus Flow
```
User opens app → Google Sign-In → App gets ID token
    ↓
Background geolocation tracks location
    ↓
POST /hubspot/status/ (email, lat, lng, api_key)
    ↓
Backend updates HubSpot CRM with student location status
```

### THINKWallet Flow
```
User opens app → Google Sign-In → Firebase verifies → Firebase ID token
    ↓
POST /users/auth/google/ { id_token } → JWT access + refresh tokens
    ↓
Authenticated API calls:
  • GET /transactions/ — view history
  • POST /transactions/ — create expense/return
  • POST /transactions/{id}/approve/ — advisor approval
  • POST /storage/upload/ → S3 presigned URL → upload receipt
  • GET /categories/ — GL/Program codes
  • GET /users/ — student/staff directory
  • GET /ledgers/stats/ — balance summary
```

---

## Deployment Architecture

| Component | Platform | URL / Notes |
|-----------|----------|-------------|
| THINK-Wallet-backend | **Heroku** (primary) | `https://think-wallet-f490339730c6.herokuapp.com` |
| THINK-Wallet-backend | Render (fallback) | `https://tgswallet-backend.onrender.com` |
| THINK-Wallet-prod | iOS App Store / TestFlight | Built via Expo EAS |
| THINKStatus-ReactNative | iOS App Store / TestFlight | Built via React Native CLI |
| THINKapps-certificates | GitHub (private) | fastlane match encryption repo |
| PostgreSQL | Heroku Postgres | Configured via `DATABASE_URL` |
| AWS S3 | `think-wallet` bucket | Receipt image storage |
| SendGrid | Email API | Transaction notifications |
| HubSpot | CRM API | Status/location tracking |
| Firebase | Auth | Google OAuth token verification |

---

## Environment / Config Requirements

### THINKStatus-ReactNative
- Node.js ≥ 18
- Yarn
- Xcode (iOS) / Android Studio (Android)
- `.env`: `API_KEY`, `BASE_URL`
- CocoaPods (`cd ios && pod install`)

### THINK-Wallet-backend
- Python 3.10+
- PostgreSQL
- `.env`: `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS`, `POSTGRES_*`, `FIREBASE_SERVICE_ACCOUNT`, `S3_BUCKET_NAME`, `SENDGRID_API_KEY`, `HUBSPOT_ACCESS_TOKEN`, `STATUS_API_KEY`, `BASE_URL`, `PLATFORM`
- Heroku CLI (for deployment)
- Run `python manage.py migrate` after each deploy (Heroku does not auto-migrate)

### THINK-Wallet-prod
- Node.js ≥ 16
- Expo CLI
- `.env` (from `env.example`): `API_BASE_URL`, `GOOGLE_IOS_CLIENT_ID`, `APP_ENV`, `DEBUG_MODE`
- `GoogleService-Info.plist` (iOS Firebase config)
- EAS Build for production builds (`eas.json` configured)

### THINKapps-certificates
- fastlane (`brew install fastlane`)
- Xcode Command Line Tools
- Match passphrase (shared with team) for encrypted certificate access

---

## Other Repos in the Organization

| Repo | Description | Status |
|------|-------------|--------|
| `utilitybelt` | TGS Staff Utility Belt | Private, older tooling |
| `group_tools` | Elgg Group plugin fork | Public fork, legacy |
| `Tidypics` | Elgg photo gallery fork | Public fork, legacy |
| `Elgg` | Social networking engine fork | Public fork, legacy |
| `Elgg-takeout` | Elgg test/dev environment fork | Public fork, legacy |
| `tabbed_profile` | Elgg profile plugin fork | Public fork, legacy |
| `mentions` | Elgg @mentions plugin fork | Public fork, legacy |

The Elgg-related repos appear to be legacy forks from an older web platform and are not part of the current active ecosystem.

---

## Authentication Flow Summary

Both apps use **Google Sign-In** restricted to `@thinkglobalschool.com` domain:

1. **THINKStatus**: Google Sign-In → ID token stored in Redux → sent as `api_key` with status updates (simpler auth model, shared API key)
2. **THINKWallet**: Google Sign-In → Firebase verifies ID token → backend exchanges for JWT (access + refresh) → JWT used for all API calls → token refresh via `/users/auth/refresh/`

---

## Key Integration Points

- **THINKStatus → THINK-Wallet-backend**: `POST /hubspot/status/` (location data → HubSpot)
- **THINKWallet → THINK-Wallet-backend**: Full REST API (JWT-authenticated)
- **THINKWallet-backend → AWS S3**: Presigned URL upload flow for receipts
- **THINKWallet-backend → SendGrid**: Transaction notification emails
- **THINKWallet-backend → HubSpot**: CRM sync for status data
- **THINKWallet-backend → Firebase**: ID token verification for Google auth
- **THINKapps-certificates → Both iOS apps**: Code signing for development, ad hoc, and distribution builds
