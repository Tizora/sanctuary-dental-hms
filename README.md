# Sanctuary Dental Hospital Management System (HMS)

**Repo goal:** a production-ready web application for Sanctuary Dental Clinic (Lilongwe, Malawi) to manage patients, appointments, billing (including scheme handling like MASM), reminders (6-month checkups), and a dental chart UI that shows teeth and surfaces worked on.

---

## Overview

Features to implement (MVP → v1 → v2):

**MVP (minimum viable product)**

* Patient registration and profiles (contacts, ID, DOB, schemes/insurer, emergency contact)
* Appointment booking (staff & patient views) with reminders
* Simple billing engine: charge items based on tooth surface(s) worked on and procedures
* Payments: cash and scheme verification (record claims/authorizations)
* 6-month automated reminders for checkups
* Dental chart UI showing teeth and surfaces (click-to-mark treated surfaces)
* Admin dashboard: daily appointments, outstanding bills, quick patient search
* Basic user auth + role-based access (admin, dentist, receptionist)

**v1 (after MVP)**

* Integration with MASM/other scheme tariff rules (configurable tariffs and limits)
* Detailed invoice generation and PDF export
* SMS & email notifications for reminders and appointment confirmations
* Reporting: revenue, appointments by provider, outstanding balances

**v2 (advanced)**

* Online patient portal for booking and viewing invoices
* Insurance claim submission & tracking (electronic or printable claim forms)
* Inventory for dental materials
* Analytics and custom report builder
* Audit logs and GDPR-like data features (export/delete patient data)

---

## Suggested Tech Stack

**Backend:** Django + Django REST Framework (Python) or Node.js + NestJS/Express. *Recommendation:* Django with PostgreSQL for built-in admin, fast modelling, and mature auth.

**Frontend:** React (create-react-app or Vite) with a component library (Tailwind CSS + Headless UI or Chakra UI). Use a dental-chart canvas component (custom SVG).

**Database:** PostgreSQL (hosted on a managed DB like ElephantSQL or Supabase)

**Notifications:** Twilio (SMS) or local SMS gateway + SendGrid or SMTP for email

**Hosting/CI:** GitHub Actions for CI; host frontend on Vercel/Netlify and backend on Render/Heroku/DigitalOcean App Platform.

**Billing & PDF:** WeasyPrint or wkhtmltopdf for PDFs, or a library (ReportLab) for Django.

---

## Key Models / Database Schema (high level)

Tables (core):

* `users` (id, name, email, role, password_hash, phone, created_at)
* `patients` (id, first_name, last_name, dob, sex, phone, email, address, national_id, scheme_id, scheme_number, notes)
* `schemes` (id, name, provider, tariff_rules, annual_limit_mk, contact_info)
* `providers` (id, name, role (Dentist/Hygienist), license_no)
* `appointments` (id, patient_id, provider_id, start_time, end_time, status, notes, created_by)
* `teeth` (id, patient_id, tooth_number (FDI or Universal), surface_state JSON, last_updated)
* `procedures` (id, code, name, description, base_price_mk)
* `procedure_items` (id, appointment_id, tooth_number, surfaces (array), procedure_id, price_mk)
* `invoices` (id, patient_id, appointment_id nullable, total_mk, status (unpaid/paid/claimed), created_at)
* `payments` (id, invoice_id, amount_mk, method (cash/cheque/scheme), scheme_authorization, paid_at)
* `reminders` (id, patient_id, type, scheduled_at, sent_at, status)

Notes:

* `teeth.surface_state` could be a JSON structure mapping surfaces (`M`,`D`,`O`,`B`,`L`,`I`) to statuses (filled, extracted, crown, etc.).
* Tariff rules for schemes should be stored in configurable JSON or separate `scheme_tariffs` table.

---

## Billing logic (teeth surface based)

Approach:

1. Define standard procedures (e.g., `Filling — 1 surface`, `Filling — 2 surfaces`, `Extraction`, `Scaling`, `Root Canal`) with pricing rules.
2. When a dentist records treatment, they select tooth number + surfaces treated. The system matches surfaces count to procedure pricing, or allows manual override.
3. Support compound pricing: base price + per-surface multiplier + material costs.
4. When patient is on a scheme (e.g., MASM) apply the scheme's tariff/benefit rules and limits before finalizing invoice.

Example calculation (pseudocode):

```
procedure = getProcedure(procedure_id)
surfaces = count(selected_surfaces)
price = procedure.base_price + (procedure.per_surface * surfaces) + materials_cost
if patient.scheme:
	price = applySchemeRules(patient.scheme, procedure, price)
```

---

## Dental Chart UI (UX notes)

* Use SVG representing 32 teeth (or 20 for deciduous). Each tooth is clickable and shows 5–6 surface zones.
* Dentist clicks a tooth -> a surface picker pops up -> choose procedure and mark as `treated` with notes.
* Color-code surfaces: filled, decayed, crowned, extracted.
* Persist each action to `procedure_items` and update `teeth.surface_state`.
* Allow history / timeline per tooth.

---

## API Endpoints (examples)

**Auth**

* `POST /api/auth/login` — returns JWT
* `POST /api/auth/register` — admin only

**Patients**

* `GET /api/patients` — list with search & filters
* `POST /api/patients` — create
* `GET /api/patients/:id` — details
* `PUT /api/patients/:id` — update

**Appointments**

* `GET /api/appointments?date=YYYY-MM-DD` — daily view
* `POST /api/appointments` — create
* `PUT /api/appointments/:id` — update status
* `POST /api/appointments/:id/cancel` — cancel and notify

**Dental chart & procedures**

* `GET /api/patients/:id/teeth` — returns tooth surfaces
* `POST /api/appointments/:id/procedures` — add procedure item (with tooth & surfaces)

**Billing**

* `POST /api/invoices` — create invoice from appointment
* `GET /api/invoices/:id/pdf` — download invoice
* `POST /api/payments` — record payment

**Reminders**

* `GET /api/reminders/due` — list pending
* `POST /api/reminders/send` — send reminders (internal/cron)

---

## Notifications & Reminders

* Use a background worker (Celery for Django or Bull for Node) to schedule reminders.
* Default reminder: 6 months after `last_checkup` or procedure; configurable per patient.
* Reminder channels: SMS & Email. Store audit of sent reminders.

---

## Repo Structure (suggested)

```
sanctuary-dental-hms/
├─ backend/                 # Django project
│  ├─ sanctuary/            # Django settings + urls
│  ├─ patients/
│  ├─ appointments/
│  ├─ billing/
│  ├─ procedures/
│  ├─ notifications/
│  └─ requirements.txt
├─ frontend/                # React app
│  ├─ src/
│  ├─ public/
│  └─ package.json
├─ infra/                   # deployment, Dockerfiles, kubernetes manifests
├─ docs/                    # ER diagrams, wireframes
└─ README.md
```

---

## Initial GitHub Issues & Milestones

**Milestone: Project Setup**

* Issue: Create GitHub repo, add MIT license
* Issue: Setup backend skeleton (Django, Dockerfile, initial models)
* Issue: Setup frontend skeleton (React, routing, auth)
* Issue: CI pipeline (GitHub Actions) for tests & lint

**Milestone: MVP**

* Issue: Patient CRUD and search
* Issue: Appointment booking + calendar view
* Issue: Dental chart component (clickable SVG)
* Issue: Billing engine basic + invoice generation
* Issue: Payments recording (cash & scheme flag)
* Issue: Reminder scheduler & sample SMS integration

---

## Initial setup commands (example)

```bash
# create repo locally
mkdir sanctuary-dental-hms && cd sanctuary-dental-hms
# initialize git
git init
# backend scaffold (Django)
python -m venv .venv
source .venv/bin/activate
pip install django djangorestframework psycopg2-binary celery redis
django-admin startproject sanctuary backend
# frontend scaffold
npx create-react-app frontend

# first commit
git add .
git commit -m "Initial scaffold: Django backend + React frontend"
# push to GitHub (create repo on GitHub web UI first)
git remote add origin git@github.com:YOUR_ORG/sanctuary-dental-hms.git
git branch -M main
git push -u origin main
```

---

## Security & Compliance notes

* Hash passwords with a modern algorithm (bcrypt/argon2). Use Django's auth by default.
* Use TLS/HTTPS for all production traffic.
* Limit access to patient PHI; use role-based permissions.
* Data backups: daily DB backups and monthly offsite backups.

---

## Next steps (what I'll do if you want me to proceed)

1. Create the GitHub repository and push the initial scaffold (Django backend + React frontend).
2. Implement the `patients` model + API + frontend CRUD pages.
3. Implement the appointment system and calendar UI.
4. Build the dental chart component and the procedure flow.

---

## Useful Malawi-specific references (for implementing schemes & tariffs)

* MASM official site for scheme rules and tariffs — keep configurable per scheme. (MASM publishes benefit brochures and tariff rules.)

---

If you want, I can:

* create the GitHub repo with the scaffold and push an initial commit (I will provide the exact commands and file templates here), or
* generate starter code files (models, serializers, basic React pages) directly into the repo as separate files in this project.

Tell me which of the two you want me to do now and I will create the files and commit-ready content.

