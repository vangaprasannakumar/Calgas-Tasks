# CALGAS Capacitors — Calgas Tasks

An internal task manager: create a task, name who's responsible for doing it and who's monitoring/helping, and track it to closed — with points earned on completion. Built entirely on Google Sheets and Google Apps Script, with an installable PWA frontend — no external hosting, database, or server required. The spreadsheet *is* the database. Same architecture and visual identity as `Stock_Manager`.

**Version:** 2.0.0 — reframed from a help-ticket desk into a task manager: multi-person assignment, a points ledger, and a fix for attachment previews not rendering inline.

---

## What it does

- Anyone creates a task with a subject, description, priority, an optional points value, one or more **Responsible people** (who's accountable for getting it done), and one or more **Monitoring people** (who oversees it and helps the responsible people).
- Either field can hold multiple people — picked from an Excel-filter-style checklist (search, select, "Select all"/"Clear").
- Either a task or any comment on it can carry **one photo attachment**, resized and compressed client-side before upload, previewed inline in the app.
- Everyone can see and comment on every task — there's no per-person visibility wall.
- Any Responsible or Monitoring person, or an Admin, can move a task forward: Open → In Progress → Completed → Closed, with reopen from Completed.
- **Points**: while a task is active, its point value sits as "pending" on every Responsible person. The moment the task is marked Completed, that value moves to "earned." Reopening reverses it; closing or cancelling a task that was never completed just drops the pending amount.
- Admins have full control: edit any task's details or reassignment, override status to anything (including Cancel), and add/edit/deactivate staff directly in the app — no spreadsheet editing needed.
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
        └─────────────────────────►│ Calgas_Tasks (the Sheet) │
                                    │ (the actual data store)   │
                                    └──────────────────────────┘
```

- **`index.html`** — single-file PWA, vanilla JS, no build step, no framework. Hosted as static files, completely separate from the Apps Script project.
- **`Code.gs`** — bound to the spreadsheet (*Extensions → Apps Script* inside the Sheet itself), deployed as a Web App. The only thing that ever touches the spreadsheet directly.
- All communication is a single `doPost` endpoint accepting a JSON body (posted as `text/plain` to sidestep Apps Script's lack of CORS preflight support) with an `action` field that routes to the right handler.
- **`sw.js`** caches the app shell for fast repeat loads and offline resilience; every request to the backend is always network-only.
- This app has its **own** login — it does not share sessions with Stock Management, Manufacturing Tracker, or ELE Tracker.

---

## Project structure

| File | Purpose |
|---|---|
| `Code.gs` | The entire backend: auth, sessions, tasks, comments/status timeline, points ledger, admin tools. |
| `index.html` | The entire frontend: login/auth screens, dashboard, task list, task detail, Team & Access. |
| `manifest.json` | PWA manifest (name, icons, theme colors) — reuses the CALGAS Capacitors logo. |
| `sw.js` | Service worker — app-shell caching, network-only for API calls. |

---

## Spreadsheet structure

**Tab names must match exactly (case-sensitive).** `Users`, `Sessions`, `PasswordResets`, `AuditLog`, and `Settings` are unchanged from the earlier help-desk version. The former `Tickets`/`Ticket_Updates` tabs are renamed to `Tasks` and `Task_Updates`.

| Sheet | Purpose |
|---|---|
| `Tasks` | One row per task: subject, description, priority, points, raised by, ResponsiblePerson, MonitoringPerson (both comma-separated usernames — a task can have several of each), status, completion notes, `AttachmentFileID`. |
| `Task_Updates` | Append-only log of every comment and status change per task, each with its own optional `AttachmentFileID` — powers the Activity timeline. |
| `Users` | Accounts: name, username, salted password hash, role, department, email, active flag, `PendingPoints`, `EarnedPoints`. |
| `Sessions` | Active login tokens (swept nightly). |
| `PasswordResets` | Pending OTP codes for password resets (swept nightly). |
| `AuditLog` | Every task edit, status override, and admin action. |
| `Settings` | Key-value config — `APP_SECRET` (password hashing/session signing), `NEXT_TASK_SEQ` (task ID counter), `ATTACHMENTS_FOLDER_ID` (set automatically on first upload). |

Columns are read by **header name**, not position — you can freely reorder or widen columns in the sheet without touching the script.

Multi-person fields (`ResponsiblePerson`, `MonitoringPerson`) store a plain comma-separated list of usernames in one cell (e.g. `nagesh,prasanna`) — there's no join table. The app reads/writes that format for you; editing it by hand in the sheet works too as long as you keep it comma-separated with valid usernames.

Uploaded photos live in their own Drive folder (named "Calgas Tasks Attachments", created automatically on first upload) rather than in the spreadsheet itself — only the file's Drive ID is stored in the sheet, and the app builds both an inline-preview URL and a full-viewer URL from that ID at display time.

---

## Roles & permissions

| | Admin | Staff |
|---|:---:|:---:|
| Create a task | ✅ | ✅ |
| View every task | ✅ | ✅ |
| Comment on any task | ✅ | ✅ |
| Move status forward (Open → In Progress → Completed) | ✅ (any task) | ✅ (only if Responsible or Monitoring person) |
| Reopen a Completed task | ✅ | ✅ (only if Responsible or Monitoring person) |
| Edit subject/description/priority/points/reassign | ✅ | ❌ |
| Cancel a task | ✅ | ❌ |
| Add / edit / deactivate people | ✅ | ❌ |

Every permission is enforced **server-side** in `Code.gs` — the frontend hiding a button is a convenience, not the actual security boundary.

---

## Setup

### If you're migrating from the earlier Help Desk / Tickets version

You've already renamed your `Tickets`/`Ticket_Updates` tabs to `Tasks`/`Task_Updates` and cleared out the old rows. From the Apps Script editor:

1. Paste this `Code.gs` over the old one.
2. Run `ADMIN_migrateTaskSchema()` once — it (re)writes the header row on `Tasks` and `Task_Updates` to the new schema. It only touches a sheet if there are **no data rows below the header**, so it's safe here but will refuse (and just log a message) if it ever finds real task rows, to avoid silently wiping data.
3. Run `ADMIN_addPointsColumns()` once — adds `PendingPoints`/`EarnedPoints` to `Users`, defaulting existing accounts to 0/0.
4. Re-deploy: **Deploy → Manage deployments → Edit → New version → Deploy.**
5. Replace `index.html`, `manifest.json`, and `sw.js` wherever you're hosting them.

Your `APP_SECRET` and existing accounts are untouched — nobody needs to reset their password.

### Fresh install

1. Create a new Google Sheet, e.g. "Calgas_Tasks". Add two empty tabs named exactly `Tasks` and `Task_Updates` (or just let step 3 create them).
2. **Extensions → Apps Script**, delete the default `Code.gs` content, paste in this project's `Code.gs`.
3. Run `ADMIN_setupSheets()` once — creates `Users`/`Sessions`/`PasswordResets`/`AuditLog`/`Settings` if missing.
4. Run `ADMIN_migrateTaskSchema()` once — creates/headers `Tasks` and `Task_Updates`.
5. Run `ADMIN_generateAppSecret()` once. Without this, login fails with an explicit error telling you to run it.
6. Run `ADMIN_createFirstAdmin('yourusername', 'Your Name', 'you@company.com', 'a-temporary-password')` once, from the editor — fill in real values first. This is your first Admin account.
7. **Deploy → New deployment → Web app.** Execute as *Me*, accessible to *Anyone* (the app handles its own auth on top of this). Copy the deployment URL.
8. Run `ADMIN_installNightlyCleanupTrigger()` once — sweeps expired sessions and OTP codes nightly.
9. In `index.html`, set `GOOGLE_API_URL` (near the top of the `<script>` block) to the Web App URL from step 7.
10. Host `index.html`, `manifest.json`, and `sw.js` together as static files — GitHub Pages, Firebase Hosting, wherever.
11. Open the hosted URL, sign in as the Admin account you created, and change the temporary password via **Forgot password**.

### Ongoing accounts

New people are added from **Team & Access** in the app — that flow emails them a temporary password automatically. `ADMIN_setUserPassword('username','new-password')` remains available in the editor as a break-glass fallback if someone's email is unreachable.

---

## Notable design decisions

- **Two separate people-fields, each multi-select** — Responsible (accountable for doing the work) and Monitoring (oversees and helps) — since real tasks often need more than one of each.
- **Points are a ledger, not a static number.** A task's point value moves through three states as its status changes — pending → earned, or pending → nothing (if closed/cancelled without completion), or earned → pending (if reopened) — computed server-side on every status transition so the numbers can't drift out of sync with what's actually on the task.
- **Editing a still-active task's points or Responsible list reconciles pending points** (removes the old amount from the old people, adds the new amount to the new people). Once a task is Completed/Closed/Cancelled, further edits no longer touch the ledger — only status transitions do, to keep the semantics simple and avoid retroactive surprises.
- **Attachment links now store a Drive file ID, not a URL.** The earlier version stored `drive.google.com/uc?export=view&...` directly, which turned out to be unreliable for inline `<img>` rendering — Drive would sometimes serve an interstitial page instead of the image, so the picture silently failed to display and fell back to showing its alt text as a plain link. Storing just the ID and building the preview URL (`drive.google.com/thumbnail?...`) and the full-view URL (`drive.google.com/file/d/.../view`) at display time fixed that, and means the embedding approach can change again later without another data migration.
- **Attachment links are "anyone with the link can view," not access-controlled.** There's no proxy in front of Drive checking someone's login before serving the image — the `<img>` tag just hits a public Drive URL directly. The link itself is only ever surfaced inside task data shown to signed-in people, but someone who obtained a link some other way could view that one image without logging in. Reasonable for low-sensitivity internal photos; worth revisiting if that ever changes.
- **Header-driven sheet access** — every read/write goes through the sheet's header row, not hardcoded column numbers, so columns can be freely reordered later without breaking anything (same convention as the Data tracker script).

---

## Known limitations / not yet done

- **One photo per task, one per comment** — not multiple attachments, and no non-image file types (PDFs, docs) yet.
- **No category/department field** — tasks route straight to people rather than through a category list.
- **Points are awarded to Responsible people only**, not Monitoring people — a task's full point value goes to *each* Responsible person (not split between them if there's more than one). Tell me if you'd rather it split, or if Monitoring people should earn a share too.
- **`MailApp` send quota** applies to reset codes and new-account emails — a plain Gmail account gets ~100 sends/day, plenty for normal use.
- **No email notifications on task activity** (e.g. "you were assigned a task") — everything is pull-based (check the app), not push. Could be added as a follow-up if useful.
