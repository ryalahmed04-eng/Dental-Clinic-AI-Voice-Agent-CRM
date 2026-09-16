# Aurelius — AI Voice Receptionist for Dental Clinics

An AI phone receptionist for a single-dentist clinic, built with **Vapi** (voice AI), **n8n** (backend workflow orchestration), **Supabase/Postgres** (data), and **Google Calendar/Gmail** (scheduling + notifications) — plus a front-desk dashboard for the dentist to manage appointments and availability.

Callers can book, cancel, or reschedule an appointment entirely by talking to the AI receptionist. The dentist gets a dashboard to view call history, manage appointments, and set weekly working hours.

![Login screen](docs/screenshots/login.png)

---

## How it works

```
Caller ──phone call──▶ Vapi Assistant (GPT-4.1)
                            │ tool calls (function calling)
                            ▼
                    n8n webhooks (7 workflows)
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
         Supabase/DB   Google Calendar   Gmail
              ▲
              │ REST (fetch)
              │
      Dashboard (static HTML/JS) ── dentist logs in, manages
      appointments, sets availability, reviews call history
```

1. A caller dials the clinic number, routed to a **Vapi** assistant.
2. Vapi's LLM asks why the caller is calling, then uses **function/tool calls** to hit n8n webhooks to check availability and book/cancel/reschedule appointments.
3. n8n workflows validate the request against the clinic's configured weekly hours, check Google Calendar for conflicts, write to Supabase, create/update/delete the calendar event, and send confirmation emails.
4. Every call is logged (caller, reason, outcome, appointment) for the dentist to review.
5. A static front-desk dashboard (plain HTML/CSS/JS, no build step) lets the dentist view appointments, edit weekly availability, and browse call history.

## Tech stack

- **Voice AI**: [Vapi](https://vapi.ai) — GPT-4.1 as the assistant's LLM, function calling for tool use
- **Backend**: [n8n](https://n8n.io) (self-hosted), 7 workflows
- **Database**: Supabase (Postgres)
- **Calendar/Email**: Google Calendar API, Gmail
- **Frontend**: Static HTML/CSS/JS (Tailwind via CDN), no build step, no framework

## Repo structure

```
backend/n8n-workflows/   7 exported n8n workflow JSON files (import into n8n)
frontend/                Static dashboard (login, dashboard, appointments, availability, call history)
docs/screenshots/        Dashboard + n8n canvas screenshots
```

## Screenshots

| Dashboard | Set Availability | Call History |
|---|---|---|
| ![Dashboard](docs/screenshots/dashboard.png) | ![Availability](docs/screenshots/set-availability.png) | ![Call History](docs/screenshots/call-history.png) |

n8n workflow canvases:

![Manage appointments workflow](docs/screenshots/n8n-manage-appointments.png)
![Book/cancel/reschedule workflow](docs/screenshots/n8n-book-appointment.png)

---

## Backend: n8n workflows

| File | Webhook path | Called by | Purpose |
|---|---|---|---|
| `N8N1-check-availability.json` | `/webhook/checkavailability` | Vapi | Checks weekly hours + Google Calendar, returns available or up to 3 suggested alternative slots |
| `N8N2-book-cancel-reschedule.json` | `/webhook/bookappointments` | Vapi | Books, cancels, or reschedules (routed by an `action` field), updates Supabase + Google Calendar, sends emails |
| `N8N3-log-call.json` | `/webhook/logcall` | Vapi | Logs every call (outcome, reason, appointment if any) |
| `N8N4-get-appointments.json` | `/webhook/getappointments` | Dashboard | Fetches appointments for the dashboard/appointments pages |
| `N8N5-availability-get-set.json` | `/webhook/availability` | Dashboard | Gets/sets the weekly working-hours table |
| `N8N6-get-calls.json` | `/webhook/getcalls` | Dashboard | Fetches call history |
| `N8N7-manage-appointments-dashboard.json` | `/webhook/manageappointment` | Dashboard | Dentist-side create/update/cancel/hard-delete of an appointment |

N8N1–N8N3 also contain the (sanitized) Vapi assistant config — system prompt and tool definitions — since they were used to create/update the assistant via the Vapi API.

## Frontend

Static HTML/CSS/JS, Tailwind via CDN, no build step, no framework.

| File | Purpose |
|---|---|
| `login.html` | Entry point — simple single-user login gate (client-side only, see caveat below) |
| `dashboard.html` | Stat cards (total/confirmed/rescheduled/cancelled) + upcoming appointments list |
| `appointments.html` | Full appointment table — search, filter by status, edit/reschedule/cancel/permanently delete |
| `availability.html` | Edit weekly working hours per weekday, shows Google Calendar sync status |
| `history.html` | Full call log from the voice agent — filter by outcome (booked/rejected/cancelled/rescheduled) |
| `sidebar.js` | Shared collapsible + searchable nav bar, included on every page |
| `aurelius.css` | Shared "obsidian & gold" theme, includes light/dark mode toggle styling |

## Database (Supabase / Postgres)

Three tables, single-dentist (no multi-tenant / no clinic_id column).

**`voice_agent_appointments`** — booked appointments (from a caller or from the dashboard)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid/serial (PK) | |
| `patient_name` | text | |
| `patient_phone` | text | caller's number, from Vapi call metadata |
| `patient_email` | text, nullable | confirmation email only sent if present |
| `appointment_date` | date | |
| `appointment_time` | time | stored `HH:MM:SS` |
| `status` | text | `confirmed` / `cancelled` / `rescheduled` |
| `google_event_id` | text | linked calendar event, used on cancel/reschedule |
| `created_at` | timestamptz | |

**`voice_agent_calls`** — log of every call, regardless of outcome
| Column | Type | Notes |
|---|---|---|
| `id` | uuid/serial (PK) | |
| `caller_phone` | text | |
| `reason` | text | free-text reason the caller gave |
| `outcome` | text | `booked` / `rejected` / `cancelled` / `rescheduled` / other |
| `appointment_date` / `appointment_time` | date / time, nullable | if an appointment resulted |
| `created_at` | timestamptz | |

**`dentist_availability`** — weekly working-hours template edited from "Set Availability"
| Column | Type | Notes |
|---|---|---|
| `weekday` | text/int | `Monday`...`Sunday` |
| `is_open` | boolean | |
| `start_time` / `end_time` | time | |

Default seed: Mon–Fri 09:00–17:00 open, Sat 09:00–13:00 open, Sun closed.

How the tables interact: `N8N1` checks `dentist_availability` first, then Google Calendar, before returning available/suggested slots. `N8N2` writes/updates `voice_agent_appointments` and mirrors changes to the calendar; on reschedule it pulls `patient_name`/`patient_email` from the matched existing row (not the fresh call payload, which won't include them). Every call gets a `voice_agent_calls` row via `N8N3`, independent of outcome. Dashboard edits (`N8N7`) touch `voice_agent_appointments` + calendar directly and don't create a call-log row.

## Vapi assistant configuration

Single Vapi assistant, GPT-4.1, function calling wired to the three caller-facing webhooks above.

**System prompt, key points:**
1. Greet the caller and ask the reason for the call; determine if it's dental-related before doing anything else.
2. **Date anchoring** — the prompt explicitly tells the model to resolve relative dates ("tomorrow", "next Tuesday") against the current date/time provided at call time, and never infer a past year. (Without this, the model would sometimes resolve dates against a stale year, corrupting bookings.)
3. Always call `check_availability` before `book_appointment`.
4. Collect required info before booking: name, phone (from call metadata), reason, date/time, optional email.
5. If the slot isn't available, offer up to 3 suggested alternatives (same day first, then forward up to 6 days), each with a spoken label like "later today" or "Thursday, September 3".
6. Reschedule/cancel by looking up the caller's existing appointment by phone number, rather than re-asking for all details.

**Tools → webhooks:**
| Tool | Webhook | Purpose |
|---|---|---|
| `check_availability` | `/webhook/checkavailability` | Availability check + suggestions |
| `book_appointment` (with `action`: book/cancel/reschedule) | `/webhook/bookappointments` | Create/cancel/update appointment |
| `log_call` | `/webhook/logcall` | Log the call outcome |

Vapi nests function-call arguments inside `message.toolCalls[0].function.arguments` (a JSON string) — each workflow's first real step is a Code node that unwraps this before anything else runs.

---

## ⚠️ Before you deploy this yourself — read this

This repo is a sanitized **portfolio/reference project**, not a drop-in production system. Before running it for real:

- **n8n credentials** (Google Calendar, Gmail, Supabase) are *not* included — n8n never exports secret values, only credential references. Reconnect every credential node to your own accounts after importing.
- **Webhook base URL**: all frontend files and the Vapi tool configs embedded in the workflow JSON use a placeholder host (`https://YOUR_N8N_INSTANCE.example.com/webhook/...`). Replace with your own n8n instance URL everywhere it appears.
- **Dashboard login**: `frontend/login.html` ships with `CHANGE_ME` / `CHANGE_ME` placeholders. This is a **client-side-only, `localStorage`-based check** — fine for a single trusted front-desk device, but it is not real authentication. Don't use this pattern for anything with real patient data without adding proper server-side auth.
- **Calendar/email addresses**: workflow JSON uses placeholders (`doctor@example.com`, `notifications@example.com`, `your_calendar_id@group.calendar.google.com`) — replace with your own.
- **Webhook security**: n8n webhooks are publicly reachable by anyone with the URL by default. Add a header/auth check on each webhook node before going live so random people can't hit your booking endpoints directly.
- No real patient data, phone numbers, or clinic infrastructure is present anywhere in this repo — everything above is a placeholder you must fill in.

## Setup order

1. Create the three tables above in Supabase (or any Postgres) and seed `dentist_availability`.
2. Import all 7 workflow files from `backend/n8n-workflows/` into n8n, reconnect credentials, replace the placeholder emails/calendar ID, and activate all workflows. Add webhook auth before going live.
3. Create the Vapi assistant per the config above, pointing its tools at your n8n webhook URLs.
4. In `frontend/*.html`, replace every `YOUR_N8N_INSTANCE.example.com` with your real n8n host, and set your own login credentials in `login.html`. Open `login.html` or host the folder on any static host.

## License

MIT — see [LICENSE](LICENSE). Personal/portfolio project; use at your own risk, especially around the auth and webhook-security caveats above.
