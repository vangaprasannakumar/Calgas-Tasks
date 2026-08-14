# CALGAS Capacitors — Help Desk

An internal help-ticket system: raise an issue, name who's responsible for it and who you're asking for help, and track it to closed. Built entirely on Google Sheets and Google Apps Script, with an installable PWA frontend — no external hosting, database, or server required. The spreadsheet *is* the database. Same architecture and visual identity as `Stock_Manager`.

**Version:** 1.0.0

---

## What it does

- Staff raise a ticket with a subject, problem description, priority, a **Responsible Person** (who's accountable for the issue/area) and an **Assigned To** person (who they're asking for help).
- Everyone can see and comment on every ticket — there's no per-person visibility wall.
- The Responsible Person, Assigned To, or an Admin can move a ticket forward: Open → In Progress → Resolved → Closed, with reopen from Resolved.
- Admins have full control: edit any ticket's details or reassignment, override status to anything (including Cancel), and add/edit/deactivate staff directly in the app — no spreadsheet editing needed.
- Installs as a PWA on desktop or mobile, with the same branded splash screen and offline-friendly shell as Stock Management.

---

## Architecture

```
┌─────────────────────┐         ┌──────────────────────────┐
│   index.html (PWA)   │  HTTPS  │   Code.gs (Apps Script)  │
│  static, hosted      │ ──────► │   bound to the Sheet,    │
│  anywhere (GitHub    │  POST   │   deployed as a Web App  │
│  Pages, Firebase,    │ ◄────── │                          │
│  Drive, etc.)         │  JSON   │                          │
└─────────────────────┘         └────────────┬─────────────┘
        │                                     │
        │ manifest.json + sw.js               │ reads/writes
        │ (installability, app-shell cache)    ▼
        │                          ┌──────────────────────────┐
        └─────────────────────────►│ Help_Desk (the Sheet)    │
                                    │ (the actual data store)   │
                                    └──────────────────────────┘
```

- **`index.html`** — single-file PWA, vanilla JS, no build step, no framework. Hosted as static files, completely separate from the Apps Script project.
- **`Code.gs`** — bound to the spreadsheet (*Extensions → Apps Script* inside the Sheet itself), deployed as a Web App. The only thing that ever touches the spreadsheet directly.
- All communication is a single `doPost` endpoint accepting a JSON body (posted as `text/plain` to sidestep Apps Script's lack of CORS preflight support) with an `action` field that routes to the right handler.
- **`sw.js`** caches the app shell for fast repeat loads and offline resilience; every request to the backend is always network-only.
- This app has its **own** login — it does not share sessions with Stock Management, Manufacturing Tracker, or ELE Tracker. Say the word if you'd rather it read Stock Management's `Users`/`Sessions` sheets instead, the way Manufacturing/ELE Tracker do.

---

## Project structure

| File | Purpose |
|---|---|
| `Code.gs` | The entire backend: auth, sessions, tickets, comments/status timeline, admin tools. |
| `index.html` | The entire frontend: login/auth screens, dashboard, ticket list, ticket detail, Team & Access. |
| `manifest.json` | PWA manifest (name, icons, theme colors) — reuses the CALGAS Capacitors logo. |
| `sw.js` | Service worker — app-shell caching, network-only for API calls. |

---

## Spreadsheet structure

| Sheet | Purpose |
|---|---|
| `Tickets` | One row per ticket: subject, description, priority, raised by, Responsible Person, Assigned To, status, resolution notes. |
| `Ticket_Updates` | Append-only log of every comment and status change per ticket — powers the Activity timeline. |
| `Users` | Accounts: name, username, salted password hash, role, department, email, active flag. |
| `Sessions` | Active login tokens (swept nightly). |
| `PasswordResets` | Pending OTP codes for password resets (swept nightly). |
| `AuditLog` | Every ticket edit, status override, and admin action. |
| `Settings` | Key-value config — `APP_SECRET` (password hashing/session signing) and `NEXT_TICKET_SEQ` (ticket ID counter). |

Columns are read by **header name**, not position — you can freely reorder or widen columns in the sheet without touching the script.

---

## Roles & permissions

| | Admin | Staff |
|---|:---:|:---:|
| Raise a ticket | ✅ | ✅ |
| View every ticket | ✅ | ✅ |
| Comment on any ticket | ✅ | ✅ |
| Move status forward (Open → In Progress → Resolved) | ✅ (any ticket) | ✅ (only if Responsible Person or Assigned To) |
| Reopen a Resolved ticket | ✅ | ✅ (only if Responsible Person or Assigned To) |
| Edit subject/description/priority/reassign | ✅ | ❌ |
| Cancel a ticket | ✅ | ❌ |
| Add / edit / deactivate people | ✅ | ❌ |

Every permission is enforced **server-side** in `Code.gs` — the frontend hiding a button is a convenience, not the actual security boundary.

---

## Setup

### 1. Spreadsheet + Apps Script

1. Create a new Google Sheet, e.g. "Help_Desk".
2. **Extensions → Apps Script**, delete the default `Code.gs` content, paste in this project's `Code.gs`.
3. Run `ADMIN_setupSheets()` once from the Apps Script editor (select it from the function dropdown, click Run). This creates all seven sheets with headers.
4. Run `ADMIN_generateAppSecret()` once. Without this, login fails with an explicit error telling you to run it.
5. Run `ADMIN_createFirstAdmin('yourusername', 'Your Name', 'you@company.com', 'a-temporary-password')` once, from the editor — fill in real values first. This is your first Admin account.
6. **Deploy → New deployment → Web app.** Execute as *Me*, accessible to *Anyone* (the app handles its own auth on top of this). Copy the deployment URL.
7. Run `ADMIN_installNightlyCleanupTrigger()` once — sweeps expired sessions and OTP codes nightly.

### 2. Frontend

1. In `index.html`, set `GOOGLE_API_URL` (near the top of the `<script>` block) to the Web App URL from step 6.
2. Host `index.html`, `manifest.json`, and `sw.js` together as static files — GitHub Pages, Firebase Hosting, wherever. They don't need to be anywhere near the Apps Script project.
3. Open the hosted URL, sign in as the Admin account you created, and change the temporary password via **Forgot password**. This also verifies the OTP email path is working — `MailApp` sends from whichever Google account the Apps Script project is authorized under.

### 3. Ongoing accounts

New people are added from **Team & Access** in the app — that flow emails them a temporary password automatically. `ADMIN_setUserPassword('username','new-password')` remains available in the editor as a break-glass fallback if someone's email is unreachable.

---

## Notable design decisions

- **Two separate people-fields on a ticket** — Responsible Person (accountable for the issue/area) and Assigned To (who's being asked to fix it) — rather than one, since they're frequently different people.
- **Everyone sees everything.** There's no category/department wall on visibility; any signed-in person can read and comment on any ticket. The only permission boundary is *changing* a ticket's status or core fields.
- **Status changes are logged as timeline entries**, not just overwritten fields — `Ticket_Updates` is append-only, so the full history of who changed what, and when, is always visible in the ticket's Activity feed.
- **Header-driven sheet access** — every read/write goes through the sheet's header row, not hardcoded column numbers, so columns can be freely reordered later without breaking anything (same convention as the Data tracker script).

---

## Known limitations / not yet done

- **No file attachments** on tickets — text only for now. Straightforward to add later via a Drive folder + attachment URL column, if wanted.
- **No category/department field** — by design, per your answer that tickets route straight to a person rather than through a category list. Easy to add a `Categories` sheet + dropdown later if that changes.
- **`MailApp` send quota** applies to reset codes and new-account emails — a plain Gmail account gets ~100 sends/day, plenty for normal use.
- **No email notifications on ticket activity** (e.g. "you were assigned a ticket") — everything is pull-based (check the app), not push. Could be added as a follow-up if useful.
