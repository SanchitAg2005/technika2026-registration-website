# Technika 6.0 — Registration Platform

Event registration and participant management platform for Technika, a multi-track technical and cultural fest.

---

## Why Technika Needed This

Technika is a large-format college technical fest running technical, creative, and cultural events simultaneously — spanning 45+ events, individual and team formats, and a significant number of registrations arriving in a short window.

Managing this through manual spreadsheets or generic form tools created specific operational problems:

- No reliable way to verify whether a participant had actually paid, or how much.
- No structured way to handle team events where members need to join after initial registration.
- No machine-readable participant data to feed into the event-day volunteer scanning system.
- Payment fraud (edited screenshots) required manual per-screenshot review.
- Duplicate UTR numbers or reused screenshots were undetectable without tooling.

The registration platform was built to address these problems end-to-end: from a participant signing up through a web UI, to their data being available to volunteers scanning QR codes at event entry points on the day.

---

## What the Platform Does

The registration platform handles the participant-facing registration flow and the administrative/operational side of managing that data.

**Participant side:**
- Gmail OTP verification before registration proceeds
- Multi-event selection (technical, creative, cultural)
- Payment UTR collection (12-digit transaction reference number)
- Payment screenshot upload
- AI-assisted payment screenshot verification (Google Gemini)
- Registration ID assignment
- Team creation and invitation flow for team events
- Dashboard to view registrations and team status
- PDF receipt generation

**Admin side:**
- Admin login and dashboard
- View all registered participants
- View all registered teams
- Google Sheets synchronization for administrative audit visibility

---

## Registration Flow

```
Participant opens registration form
         │
         ▼
  Gmail OTP verification
         │
         ▼
  Personal details + event selection
  (Name, Age, Gender, Email, WhatsApp,
   Institution, Course, Semester)
         │
         ▼
  Payment: UTR (12-digit) + screenshot upload
         │
         ▼
  Gemini AI verifies screenshot:
  ├── Transaction status: SUCCESS
  ├── Tamper/edit detection
  ├── Payee verification (ARKA JAIN UNIVERSITY)
  ├── Amount vs. selected events check
  └── UTR extraction and cross-check vs. entered UTR
         │
         ▼
  6-character Registration ID generated
  (random, uniqueness enforced via MongoDB)
         │
         ▼
  User saved to MongoDB
  Screenshot uploaded to Cloudinary
         │
         ▼
  Event registrations created
  (INDIVIDUAL or TEAM per event)
         │
         ▼
  Google Sheets sync (background, non-blocking):
  ├── Payment audit log
  └── Participant data sheet
         │
         ▼
  Registration confirmed — ID returned to participant
```

---

## Individual vs. Team Events

Events in the system are configured at the data level with two flags: `individualAllowed` and `teamAllowed`.

### Individual Events

The participant registers for the event directly. A `Registration` document is created with type `INDIVIDUAL` and status `CONFIRMED`.

### Team Events

When a participant selects a team event at registration time:
- A `Team` is automatically created with the registering participant as the **Leader**.
- A `TeamMember` record is added for the leader.
- Team status starts as `forming`.
- The leader can then invite other registered participants to join the team by email via the dashboard.
- Additional members accept the invitation and join the team.

Team IDs follow the format `T` + 5-digit number (e.g. `T10492`).

---

## Payment Verification

Payment verification is handled using **Google Gemini AI** analyzing the uploaded UPI payment screenshot.

The verification checks:
1. The transaction status is `SUCCESS` (not pending, failed, or initiated)
2. The screenshot does not show signs of digital editing or tampering
3. The payee is verified as ARKA JAIN UNIVERSITY on the correct UPI handle
4. The verified amount matches the expected amount for the selected events
5. The UTR extracted from the screenshot is compared against the manually entered UTR

If any of these checks fail, the registration is rejected with a specific error message. The participant is required to submit a valid, unedited screenshot of a completed payment before registration can proceed.

Screenshot storage:
- **Primary**: Cloudinary (with automatic image compression before upload)
- **Fallback**: Local filesystem (used during development or if Cloudinary is not configured)

---

## Payment Pricing

| Selection | Fee |
|---|---|
| Any technical/creative/cultural events | ₹150 (flat, regardless of count) |
| Paint Ball add-on | ₹350 |
| Night Show add-on | ₹650 |

The backend calculates the expected amount from the selected events and verifies the screenshot matches before proceeding.

---

## Registration IDs

Every registered participant receives a unique **6-character alphanumeric Registration ID** (uppercase letters A-Z and digits 0-9).

Example: `AB3X7K`, `TZ0021`, `R9KM4P`

IDs are generated randomly. Uniqueness is verified against MongoDB before assignment, and a unique database index enforces this constraint as a safety net.

---

## Architecture

```
Participant Browser
        │
        ▼
┌─────────────────────────────────┐
│   Technika Registration Website │
│                                 │
│   Frontend: React + TypeScript  │
│   (Vite, Tailwind CSS, React    │
│    Router, qrcode)              │
│                                 │
│   Backend: Node.js + Express    │
└─────────────┬───────────────────┘
              │
    ┌─────────┼──────────────────────┐
    │         │                      │
    ▼         ▼                      ▼
 MongoDB   Cloudinary           Gemini AI
(primary   (payment           (screenshot
   DB)    screenshots)       verification)
    │
    ├──► Google Sheets (admin audit log, participant sync)
    ├──► Gmail SMTP (OTP verification emails)
    └──► PDFKit (registration receipts)
```

### Relationship to Event-Day Services

The registration platform is **one part of the larger Technika system**. On event day, a separate backend — [`Synchroniser_technika`](https://github.com/SanchitAg2005/Synchroniser_technika) — mirrors participant data from MongoDB into a Supabase/PostgreSQL database and provides the API used by volunteer QR scanners for attendance marking, winner declaration, and leaderboard management.

```
Registration Platform        Event-Day Services
(technika2026-               (Synchroniser_technika)
registration-website)
        │                           │
    MongoDB  ──── sync ────►  Supabase/PostgreSQL
                                    │
                             Volunteer Scanner App
                             (Flutter, referenced)
```

The two systems are connected by synchronization, not a shared database. See the [Synchroniser_technika README](https://github.com/SanchitAg2005/Synchroniser_technika) for implementation details of the event-day services.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React 19, TypeScript |
| Build Tool | Vite |
| Styling | Tailwind CSS v4 |
| Routing | React Router v7 |
| Backend | Node.js, Express |
| Database | MongoDB via Mongoose |
| Payment Image Storage | Cloudinary |
| AI Verification | Google Gemini (`@google/genai`) |
| Email (OTP) | Nodemailer + Gmail SMTP |
| Admin Sync | Google Sheets API (`googleapis`) |
| Receipt Generation | PDFKit |
| QR Code (frontend) | `qrcode` |

---

## Local Development

The repository is a monorepo containing `frontend/` and `backend/` directories.

### Prerequisites

- Node.js (check `.nvmrc` or `package.json` engines if present)
- MongoDB instance (local or Atlas)
- Cloudinary account (or use without it — local fallback active)
- Google Cloud service account with Sheets API access (optional for local dev)
- Gemini API key
- Gmail app password for SMTP

### Install dependencies

```bash
# From repo root — installs both frontend and backend
npm run install:all
```

### Backend environment variables

Create `backend/.env` based on `backend/.env.example`:

```
PORT=5000
MONGODB_URI=<your MongoDB connection string>
JWT_SECRET=<your JWT secret>

CLOUDINARY_CLOUD_NAME=<your Cloudinary cloud name>
CLOUDINARY_API_KEY=<your Cloudinary API key>
CLOUDINARY_API_SECRET=<your Cloudinary API secret>
CLOUDINARY_FOLDER=technika-payment-screenshots

GEMINI_API_KEY=<your Gemini API key>

SMTP_USER=<your Gmail address>
SMTP_PASS=<your Gmail app password>

GOOGLE_SERVICE_ACCOUNT_EMAIL=<your service account email>
GOOGLE_PRIVATE_KEY=<your service account private key>
GOOGLE_SPREADSHEET_ID=<your Google Sheet ID>
```

> Do not commit real credentials. The `.env` file is gitignored.

> If `GEMINI_API_KEY` is not set, payment verification is mocked and will always return success — useful during local development.

> If Cloudinary credentials are not configured, screenshots fall back to local filesystem storage.

### Start backend

```bash
cd backend
npm run dev
```

### Start frontend

```bash
cd frontend
npm run dev
```

---

## Current Status

### Implemented and verified

- Participant registration flow (OTP → form → payment → confirmation)
- Gemini AI payment screenshot verification
- Individual event registration
- Team event creation, leadership, and invitation flow
- 6-character random Registration ID assignment
- MongoDB persistence for users, events, registrations, teams, team members, invitations
- Cloudinary payment screenshot storage (with local fallback)
- Google Sheets synchronization (participant sync + payment audit log)
- PDF receipt generation
- Admin dashboard (users view, teams view)
- Synchroniser backend microservices (sync service + volunteer/leaderboard API)

### Referenced but not locally verified

- Flutter volunteer scanner client — referenced in the Synchroniser architecture but the source repository was not available during audit.

---

## Security Notes

- Environment variables must not be committed to version control.
- Payment screenshots contain sensitive financial transaction data — Cloudinary access should be appropriately restricted.
- The Gemini API key and Google service account private key are production credentials.
- The `backend/.env.example` contains placeholder values only — never copy real credentials into documentation or commit them.

---

## Historical Repositories

Earlier versions of the Technika registration system exist as separate repositories. They represent previous iterations and should not be confused with the current platform:

| Repository | Description |
|---|---|
| [`technika_registration`](https://github.com/SanchitAg2005/technika_registration) | First iteration of the registration backend |
| [`technika_registration_v.2`](https://github.com/SanchitAg2005/technika_registration_v.2) | Second iteration of the registration backend |
| [`technika6.0`](https://github.com/SanchitAg2005/technika6.0) | Static marketing website for Technika 6.0 |

None of these are the current active registration platform.
