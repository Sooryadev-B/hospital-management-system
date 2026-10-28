# Care Hospital: three independent Django apps, one shared database

Structured like the Overclock reference (storefront + admin portal), plus a Doctor Portal: **three separate Django projects**, each with its own
server, settings, URLs, templates, static files, login and cookies, reading and writing **one shared database**.

| | Folder | Port | Purpose |
|---|---|---|---|
| **Patient Site** | `patient_site/` (app `patient_portal`) | 8000 | Patients: register, find doctors, check slots, book, track status |
| **Admin Portal** | `admin_site/` (app `hospital_admin`) | 8001 | Staff: approve/reject/reschedule, doctors, schedules, leave, reports |
| **Doctor Portal** | `doctor_site/` (app `doctor_portal`) | 8002 | Doctors: sign in with the username + password an admin assigned, see their own appointments and schedule, approve/reject/cancel/reschedule them, mark visits completed, notifications |
| **Shared core** | `shared/hospital_core/` | - | Models, migrations, availability + booking services, permissions |

```
hospital_system/
├── shared/
│   ├── db.sqlite3                  # the ONE database (created by migrate)
│   └── hospital_core/              # models, migrations, services.py, permissions.py, seed_demo
├── patient_site/                   # Project 1 (port 8000)
│   ├── manage.py
│   ├── patient_config/             # settings, urls
│   └── patient_portal/             # views, forms, templates, static, tests
├── admin_site/                     # Project 2 (port 8001)
│   ├── manage.py
│   ├── admin_config/               # settings, urls
│   └── hospital_admin/             # views, forms, templates, static, tests, setup_admin
├── doctor_site/                    # Project 3 (port 8002)
│   ├── manage.py
│   ├── doctor_config/              # settings, urls
│   └── doctor_portal/              # views, forms, templates, static, tests
└── integration_tests/              # patient -> admin -> doctor journeys across all three sites
```

## How they connect
All three `settings.py` files add `shared/` to `sys.path`, install `hospital_core`, and call `hospital_core.conf.database_config()`,
so they point at the same database (SQLite file by default, PostgreSQL with `DB_ENGINE=postgres`). A booking made on
port 8000 is a row in the shared `Appointment` table, so it shows up on port 8001 immediately, and the status the admin sets
shows up for the patient. Nothing is copied.

Because the projects can't `reverse()` each other's URLs, cross-site links (notification links, "Admin Portal" / "View patient
site", staff-sent-to-wrong-site redirects) use `PATIENT_SITE_URL` / `ADMIN_SITE_URL` / `DOCTOR_SITE_URL` from settings via `hospital_core/links.py`.
Each site's tests assert those hand-built paths still match its real URLs.

**Cookies:** browsers share cookies across ports on the same host, so each site uses its own cookie names
(`patient_sessionid` / `admin_sessionid` / `doctor_sessionid`, and likewise for CSRF). You can be logged in to all three at once.

## Setup (first time)
```bash
pip install -r requirements.txt

# Terminal 1 - Patient Site (it is fine to run migrations from either project)
cd patient_site
python manage.py makemigrations hospital_core     # first time only: generates shared/hospital_core/migrations
python manage.py migrate
python manage.py seed_demo                        # optional: demo departments, doctors, Mon-Fri schedules
python manage.py runserver 8000

# Terminal 2 - Admin Portal
cd admin_site
python manage.py setup_admin --username admin     # prompts for a password; creates the staff/superuser
python manage.py runserver 8001

# Terminal 3 - Doctor Portal (no setup needed: doctor accounts are created from the Admin Portal)
cd doctor_site
python manage.py runserver 8002
```
- Patient Site: http://127.0.0.1:8000/  (register at `/register/`, login `/login/`, dashboard `/dashboard/`)
- Admin Portal: http://127.0.0.1:8001/login/
- Doctor Portal: http://127.0.0.1:8002/login/

**After pulling this update, run `python manage.py makemigrations hospital_core` and `python manage.py migrate` once** (from any project): `Doctor` gained a nullable `user` link for the doctor's login.

## Doctors completing appointments
On the Doctor Portal a doctor can press **Mark as completed** on their own *Approved* appointment
(dashboard or appointment page). The patient is notified.

## Doctor appointment permissions
Doctors have the same appointment actions as the admin, limited to **their own** appointments (another doctor's ID returns 404): **Approve**, **Reject**,
**Cancel**, **Reschedule** (sets Approved), and **Decline patient request**, each with an optional note to the patient. All are POST + CSRF and go through
the same `services` functions as the admin, so transitions, slot checks, audit log and patient notifications behave identically.

## Doctor logins
An administrator assigns a doctor's login in the Admin Portal: **Doctors -> Add doctor** (or **Edit** for an existing doctor) -> fill in
**Login username** and **Login password**. New doctors must have both; usernames are unique. Email is optional contact info.
Existing doctors (e.g. from `seed_demo`) have no login until an admin adds a username and password while editing them. Editing a doctor and
leaving the password blank keeps the current one; entering a new one resets it. Deactivating a doctor blocks their Doctor Portal access.
Doctors can change their own password under **Profile & password**.

## Security
- One shared `auth.User` table; **admin role = `is_staff`**, only creatable through `setup_admin` / `createsuperuser`.
  **Doctor role = a user linked to a `Doctor` row**, only creatable by an admin in the Admin Portal; doctor accounts are never staff.
  The registration form has no role fields, so a patient can't become an admin or a doctor.
- Admin Portal: anonymous -> admin login; logged-in patient -> 403; admin login rejects non-staff accounts.
- Doctor Portal: completing is POST + CSRF, only for the doctor's own Approved appointments that have started. Anonymous -> doctor login; logged-in patients/admins and deactivated doctors -> 403; every appointment query is scoped to the
  signed-in doctor (another doctor's appointment ID returns 404). Doctor login rejects patient, admin and deactivated accounts. Doctor accounts
  are rejected at the patient login and the admin login, and are excluded from the Admin Portal's patient lists and counts.
- Patient Site: staff accounts are rejected at the patient login and redirected to the Admin Portal; patients only ever
  query their own appointments (other IDs return 404).
- All state changes are POST + CSRF protected; CSV export neutralises spreadsheet-formula injection.
- There is no hard-coded admin password; `setup_admin` prompts for one and runs Django's password validators.

## Availability and booking rules (`shared/hospital_core/services.py`)
Slots = weekly schedule (+ "extra hours" exceptions) minus past slots, breaks, leave, unavailable periods and slots held by
**Pending/Approved** appointments. Booking re-validates on the server inside `transaction.atomic()` (row lock on the doctor,
effective on PostgreSQL) and a partial unique DB constraint guards against double booking even across the two servers.
- New appointments are `Pending` and hold the slot; reject/cancel/expire releases it.
- **Expiry policy:** a Pending appointment whose start time passes without action becomes `Expired`
  (checked on dashboard/list loads, or run `python manage.py expire_appointments` from cron).
- Reschedule validates the new slot before the old one is released; an admin reschedule sets the status to Approved.
- Patients cancel Pending appointments directly; for Approved ones, and for reschedules, they send a request the admin accepts or declines.

## Tests
```bash
cd patient_site && python manage.py test hospital_core patient_portal   # shared models/services + patient site
cd admin_site   && python manage.py test hospital_admin                 # admin portal (incl. doctor login management)
cd doctor_site  && python manage.py test doctor_portal                  # doctor portal
cd integration_tests && python manage.py test                           # patient -> admin -> doctor journeys across all three sites
```

## Notes
- SQLite is fine for development with both servers running (20 s lock timeout); use PostgreSQL in production.
- Run all `migrate`/`makemigrations` for the shared app from one project, as in the Overclock reference.
- Environment variables: see `.env.example`.
# hospital-management-system
