# CARE OS AI

A single-page **hospital management system** with an animated AI assistant ("Mr.Doc"), backed by a small dependency-free Python server and a SQLite database.

---

## Features

| Module | What it does |
| --- | --- |
| **Dashboard** | Live stats, revenue and patient charts (Chart.js), calendar, timeline, recent activity |
| **Appointments** | Create, filter, reschedule and cancel appointments |
| **Patients** | Patient list, detailed charts (vitals, allergies, conditions, medications, clinical notes), add/edit |
| **Doctors** | Doctor directory with specialty, availability and profile details |
| **Departments** | Department overview and per-department drill-down |
| **Schedule** | Weekly schedule view with week navigation |
| **Room Allotment** | Admit and discharge patients, track wards, rooms and beds (stored in SQLite) |
| **Billing** | Invoices, mark as paid, printable invoices, billing stats |
| **Inventory** | Stock levels, low/critical alerts, restocking, item history |
| **Messages** | Staff chat threads and conversation search |
| **AI Copilot (Mr.Doc)** | Chat assistant that answers questions about patients, doctors, beds, appointments, inventory and billing; animated 2D/3D character, voice input and spoken replies |
| **Auth & UX** | Login screen, notifications, global search, light/dark theme, collapsible sidebar |

### AI Copilot

- Answers are generated from the hospital's own records (patients, beds, billing, stock, etc.). **No external AI API is called.**
- Bed and occupancy questions are answered by the server from live SQLite data (`POST /api/ai/chat`); everything else is answered in the browser from the loaded records.
- Voice input and text-to-speech use the browser's built-in Web Speech API.
- Selectable voice/greeting languages: English (US/UK), Hindi, Telugu, Tamil, Spanish, French, German.

---

## Tech stack

- **Frontend:** vanilla HTML, CSS and JavaScript (`index.html`, `assets/styles.css`, `js/app.js`), Chart.js loaded from a CDN, bundled Three.js for 3D character and anatomy models
- **Backend:** Python 3 standard library only (`http.server`, `sqlite3`), so there is nothing to `pip install`
- **Database:** SQLite (`care_os.sqlite3`, created and seeded automatically if missing)

---

## Project structure

```
care-os-ai/
├── care_os_server.py      # HTTP server, REST API and SQLite layer
├── care_os.sqlite3        # System-of-record database (auto-created)
├── index.html             # Single-page app shell (all pages/modals)
├── js/
│   └── app.js             # App logic, seed data, AI copilot, auth, voice
└── assets/
    ├── styles.css         # All styling (light/dark themes)
    ├── voice-orb.html     # Voice orb component
    ├── vendor/            # Three.js, loaders, qrcode.js
    ├── anatomy/           # Body-part images and X-ray GLB model
    ├── auth-character/    # Login-screen character states
    ├── get-started-jump/  # Welcome animation frames
    ├── rig-preview/       # Mr.Doc pose previews
    ├── mrdoc-source/      # Blender source files for Mr.Doc
    ├── mrdoc-reference/   # Character reference sheets
    └── *.glb, *.png, *.svg# Mr.Doc models, expressions and logos
```

---

## Getting started

### Requirements

- Python 3.8 or newer
- A modern browser (Chrome or Edge recommended for voice input)
- Internet access on first load for the Chart.js CDN script

### Run

```bash
cd care-os-ai
python3 care_os_server.py
```

Then open **http://localhost:8080**.

To use a different port:

```bash
PORT=3000 python3 care_os_server.py
```

### Sign in

The app opens on a login screen. The default administrator account is defined as `DEFAULT_ADMIN` in `js/app.js`, and the login page has a **"Use these"** shortcut that fills it in. The account and session are stored in the browser's `localStorage`.

> **Before any real deployment, change the default credentials.** Authentication is client-side only (see [Limitations](#limitations)).

---

## How data is stored

- **`rooms`** and **`allotments`** are normalised SQLite tables (wards, beds, admissions). The server seeds 9 rooms and a few sample admissions on first run.
- **Everything else** (patients, doctors, appointments, billing, inventory, messages, recent activity, copilot history) is stored in a `records` table as one JSON document per row, keyed by `(collection, id)`. This keeps nested data such as allergies, notes and chat threads intact without schema migrations.
- On load, the app calls `/api/bootstrap`. If the database is empty it uses its built-in sample data. After every edit, the app saves the changed collection back with `PUT /api/collections/<name>`.

To reset all data, stop the server, delete `care_os.sqlite3` and start it again.

---

## API reference

All endpoints return JSON and are served by `care_os_server.py`.

| Method | Endpoint | Description |
| --- | --- | --- |
| GET | `/api/health` | Service and database status |
| GET | `/api/bootstrap` | All collections plus bed summary in one request |
| GET | `/api/collections/<name>` | Rows of one collection |
| PUT | `/api/collections/<name>` | Replace a collection. Body: `{"rows": [...]}`. Rows missing from the payload are deleted |
| GET | `/api/rooms` | Rooms with live occupied/available beds |
| GET | `/api/allotments` | All admissions, newest first |
| POST | `/api/allotments` | Admit a patient (`patient_name` and `room_number` required) |
| POST | `/api/allotments/<id>/discharge` | Discharge an admission |
| GET | `/api/stats` | Occupancy summary (beds, rooms, active admissions) |
| POST | `/api/ai/chat` | Bed/occupancy answers from SQLite. Body: `{"message": "..."}` |

Valid collection names: `patients`, `doctors`, `appointments`, `billing`, `inventory`, `messages`, `recentActivity`, `copilot`.

Example:

```bash
curl http://localhost:8080/api/stats
curl -X POST http://localhost:8080/api/allotments \
  -H "Content-Type: application/json" \
  -d '{"patient_name":"Jane Doe","room_number":"201","bed_number":"Bed-A","ward":"Cardiology Wing"}'
```

---

## Limitations

- **Not production-ready security.** Login is checked in the browser against a hashed password in `localStorage`; the API has no authentication and allows CORS from any origin (`*`). Do not expose it to the internet or use real patient data without adding proper server-side auth, HTTPS and access controls.
- **Not a medical device.** The copilot's answers are rule-based summaries of stored records and are not clinical advice.
- **Occupancy is not enforced.** The server does not stop you from admitting more patients than a room has beds.
- **Whole-collection saves.** `PUT /api/collections/<name>` replaces the full collection, so simultaneous edits from several browsers can overwrite each other (last write wins).
- **Social sign-in, registration and password recovery** are placeholders and are not connected to any provider.
- Voice features depend on browser support for the Web Speech API.

---

## Customising

- **Seed data and copilot logic:** `js/app.js`
- **Styling and themes:** `assets/styles.css`
- **Default rooms and sample admissions:** `init_db()` in `care_os_server.py`
- **Mr.Doc character:** models and Blender sources in `assets/`
