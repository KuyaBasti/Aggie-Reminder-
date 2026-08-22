# Aggie Reminder

A volunteer-shift reminder system for **Aggie House**, built as a HackDavis 2024 hackathon project. It ships **two independent delivery paths** for the same job: a **Node.js/Express web app** (PostgreSQL + Knex for logins, `xlsx` for roster ingestion, SendGrid for email) where an admin views the shift schedule and fires off custom reminders, and a **Google Apps Script** that runs on a time-driven trigger and emails every volunteer whose shift is at most two days away — no server required.

The interesting part isn't the web stack — it's that **the spreadsheet is the database**. The shift schedule and volunteer roster never get imported anywhere: the Express server re-reads `Test Calendar 2024.xlsx` from disk on every request, and the Apps Script reads its Google Sheet in place on every trigger run. PostgreSQL exists only to remember who can log in. Both paths consume the same row shape — *Shift | Date | Volunteers | Email…* — but share zero code, which is exactly why the Apps Script path keeps working after the hackathon laptop closes.

---

## Table of Contents

1. [How a Reminder Gets Sent](#how-a-reminder-gets-sent)
2. [Repository Map](#repository-map)
3. [The Express Server](#the-express-server)
4. [The Spreadsheet Contract](#the-spreadsheet-contract)
5. [The Frontend](#the-frontend)
6. [The Apps Script Automation](#the-apps-script-automation)
7. [Setup & Run](#setup--run)
8. [API Endpoints](#api-endpoints)
9. [Known Limitations & Sharp Edges](#known-limitations--sharp-edges)
10. [Provenance](#provenance)

---

## How a Reminder Gets Sent

```mermaid
flowchart TD
    A["Admin browser —<br/>reminder form on index.html"] -- "POST /send-custom-reminders<br/>subject / message / recipients" --> S["Express server<br/>server.js, port 3000"]
    S --> R{"recipients<br/>equals 'all'?"}
    R -- "yes" --> X["readExcelData(filePath, 1) —<br/>sheet index silently ignored,<br/>reads sheet 0; item.email is<br/>undefined (header is 'Email')"]
    R -- "no" --> P["split on commas, trim —<br/>the path that works"]
    X --> L["forEach sendReminderEmail —<br/>fire and forget"]
    P --> L
    L --> SG["SendGrid @sendgrid/mail<br/>from aggiehouseschedules@gmail.com"]
    SG --> V["volunteer inboxes"]
    S -- "responds immediately" --> OK["'Emails sent successfully' —<br/>before any send resolves"]

    G["Google Sheet<br/>Shift, Date, Volunteers, Email..."] --> T["Apps Script senderEmail()<br/>time-driven day timer"]
    T --> D{"event date minus today<br/>at most 2 days?"}
    D -- "yes" --> M["MailApp.sendEmail per address —<br/>columns D onward on the row"]
    D -- "no" --> SKIP["skip row"]
    M --> V
```

The top half needs a running server, a Postgres instance, and a SendGrid key; the bottom half needs only a Google Sheet and a copied-in script. They never talk to each other.

## Repository Map

```text
Aggie-Reminder--main/
├── README.md                    # you are here
├── SYSTEM-DESIGN.md             # the architecture-level view
├── Procedures.md                # original step-by-step run guide (Node, PostgreSQL, SendGrid)
├── FILELIST.txt                 # generated file listing — an artifact, not documentation
├── automated_reminder/          # path 2: zero-server reminders via Google Apps Script
│   ├── automated_reminder.txt   # the script — paste into Code.gs, add a Day-timer trigger
│   ├── README!.txt              # 12-step setup: Sheet → Extensions → Apps Script → Triggers
│   └── Formatted Sheet.xlsx     # the sheet shape the script expects: Shift | Date | Volunteers | Email…
└── hackdavis2024/               # path 1: the Express web app
    ├── server.js                # the entire backend — routes, Knex/Postgres, xlsx ingestion, SendGrid
    ├── package.json             # express, knex, pg, xlsx, @sendgrid/mail, cors… (+ 3 stowaway deps)
    ├── package-lock.json
    ├── Test Calendar 2024.xlsx  # sample workbook: 'Schedule' (79 shift rows) + 'Contact Info'
    ├── .vscode/launch.json      # Chrome debug config aimed at :8080 — the server runs on :3000
    ├── node_modules/            # installed npm dependencies — treat as build artifacts
    └── public/
        ├── index.html           # admin dashboard: reminder form + shifts table
        ├── login.html           # login form
        ├── register.html        # registration form
        ├── css/                 # styles.css (dashboard), form.css (login/register), home.css (orphaned)
        ├── js/
        │   ├── scripts.js       # fetches /api/shifts, renders week-separated table, submits reminders
        │   ├── form.js          # login/register fetch + sessionStorage session + sliding alert box
        │   └── home.js          # client-side auth gate (+ leftovers targeting elements that don't exist)
        └── img/                 # aggie_house.png (logo & form background), bg.png
```

## The Express Server

[server.js](hackdavis2024/server.js) is the whole backend — one file, ~150 lines, five concerns:

- **Static hosting** — `express.static` serves `public/`; `/`, `/login`, and `/register` each `sendFile` their page. (The static middleware is registered twice — once with the relative `'public'`, once with the absolute `intialPath` — and JSON body parsing is registered twice too, via `express.json()` *and* `body-parser`. Harmless, but both are doubled.)
- **Auth** — `POST /register-user` inserts `{name, email, password}` into a Postgres `users` table via Knex and returns the name and email; `POST /login-user` does a `select ... where {email, password}` and returns the match or `'email or password is incorrect'`. Passwords are stored and compared as **plaintext** — this is demo-level auth.
- **Excel ingestion** — `readExcelData(filePath)` is `xlsx.readFile` → first sheet → `sheet_to_json`. The path is a **hardcoded absolute Windows path** (`C:\Users\minhk\...\Test Calendar 2024.xlsx`) that you must edit before anything works.
- **Email** — `sendReminderEmail(to, subject, text)` wraps `sgMail.send` with the fixed verified sender `aggiehouseschedules@gmail.com`. The API key is a hardcoded `const API_KEY = ''` — also edited in source.
- **The reminder endpoint** — `POST /send-custom-reminders` takes `{subject, message, recipients}`; `recipients` is either `'all'` (roster emails from the workbook) or a comma-separated list. It loops `sendReminderEmail` over the list and responds `'Emails sent successfully'` **without awaiting a single send** — failures exist only in the server console. The `'all'` branch is broken twice over; see [sharp edges](#known-limitations--sharp-edges).

The database connection is hardcoded: host `127.0.0.1`, user `postgres`, password `'2002'`, database `hackdavis2024`.

## The Spreadsheet Contract

Both delivery paths are shaped around one row layout, verified from the committed sample workbooks:

- **[Test Calendar 2024.xlsx](hackdavis2024/Test%20Calendar%202024.xlsx)** (the web app's file) has two sheets. **`Schedule`** — 79 populated rows of `Shift Time | Date | Volunteers | Email` (some rows carry a second email column; the used range is padded with blanks out to row 278), with dates as raw Excel serial numbers. **`Contact Info`** — `Name (First Last) | Preferred Pronouns | Email | Phone Number`. Only `Schedule` is ever actually read: `readExcelData` always takes `SheetNames[0]`, and the `, 1` argument the reminder endpoint passes to reach `Contact Info` doesn't exist in the function's signature.
- **[Formatted Sheet.xlsx](automated_reminder/Formatted%20Sheet.xlsx)** (the Apps Script's sheet, uploaded to Google Sheets) has a single `Sheet1`: `Shift | Date | Volunteers | Email` with additional email addresses continuing rightward — the script harvests **every cell from column D onward** on a matching row.

Because `/api/shifts` returns the raw `sheet_to_json` of `Schedule`, the JSON the browser receives includes the volunteer **email columns**, even though the table only renders Shift Time / Date / Volunteers — and the endpoint has no auth (see sharp edges).

## The Frontend

Three static pages, no framework:

- **[index.html](hackdavis2024/public/index.html)** — the dashboard: a send-reminder form (subject, message, recipients) above a shifts table. [scripts.js](hackdavis2024/public/js/scripts.js) fetches `/api/shifts`, converts each Excel serial date with `(serial − (25567 + 2)) × 86,400,000` ms (the Excel→Unix epoch offset) plus a timezone-offset correction, and inserts a bold **"Week N"** separator row after every 7 shift rows. The form POSTs to `/send-custom-reminders` and `alert()`s the response.
- **[login.html](hackdavis2024/public/login.html) / [register.html](hackdavis2024/public/register.html)** — glassy forms over the Aggie House photo. [form.js](hackdavis2024/public/js/form.js) staggers each field's fade-in by `i × 100 ms`, POSTs to `/login-user` or `/register-user`, and on success stores `name` and `email` in `sessionStorage` and redirects to `/`. Errors slide an alert box down for 5 seconds.
- **The session model is entirely client-side** — [home.js](hackdavis2024/public/js/home.js) redirects to `/login` when `sessionStorage.name` is missing. Nothing server-side checks anything after login.

## The Apps Script Automation

[automated_reminder.txt](automated_reminder/automated_reminder.txt) is a single function, `senderEmail()`, meant to be pasted into a Google Sheet's `Code.gs` and bound to a **time-driven Day timer** trigger ([README!.txt](automated_reminder/README!.txt) walks through all 12 clicks). Each run it:

1. Reads all rows of `Sheet1`, drops the header.
2. Normalizes today and each row's date (column B) to midnight, computes `diff = round((date − today) / 86,400,000)` days.
3. If `diff <= 2`, harvests every address from **column D onward** (`event.slice(3)`) and sends each one an email — subject `AggieHouse Reminder`, body `"This is a reminder for you upcomming volunteer on "` plus the date (typos included).

There is deliberately no infrastructure here: Google hosts the sheet, the scheduler, and the mailer (`MailApp`). Note the condition has **no lower bound** — see sharp edges.

## Setup & Run

Real requirements, honestly: **Node.js 16+** (the strictest `engines` floor in the dependency tree — knex 3's; `package.json` itself declares none), **any recent PostgreSQL**, a **SendGrid account with a verified sender and API key**, and hand-editing `server.js` — there is no `.env` support. (The original [Procedures.md](Procedures.md) covers the same steps in walkthrough form, including PostgreSQL installation from scratch.)

1. **Database** — create it and the one table the app uses:

   ```sql
   CREATE DATABASE hackdavis2024;
   \c hackdavis2024
   CREATE TABLE IF NOT EXISTS users (
     id SERIAL PRIMARY KEY,
     name TEXT NOT NULL,
     email TEXT UNIQUE NOT NULL,
     password TEXT NOT NULL
   );
   ```

2. **Edit [server.js](hackdavis2024/server.js)** — set `API_KEY` to your SendGrid key (line 18), `filePath` to the absolute path of your `.xlsx` (line 10), and the Knex `connection` block to your Postgres credentials (lines 21–29). The `from:` address in `sendReminderEmail` must be a sender you've verified in SendGrid.

3. **Install and run:**

   ```bash
   cd hackdavis2024
   npm install
   npm start        # runs: nodemon server.js → "listening on port 3000......"
   ```

   Open `http://localhost:3000`, register an account, and you land on the dashboard.

4. **Apps Script path** (independent of all of the above) — upload `Formatted Sheet.xlsx` to Google Sheets, paste `automated_reminder.txt` into **Extensions → Apps Script → Code.gs**, and add a **Time-driven → Day timer** trigger at your preferred hour. Running the function manually in the editor sends immediately.

## API Endpoints

| Method | Path | Body | Does |
|---|---|---|---|
| GET | `/` | — | serves `index.html` (dashboard) |
| GET | `/login`, `/register` | — | serve the auth pages |
| GET | `/api/shifts` | — | first sheet of the workbook as JSON (all columns, emails included) |
| POST | `/register-user` | `{name, email, password}` | inserts into `users`, returns `{name, email}` |
| POST | `/login-user` | `{email, password}` | exact-match lookup, returns `{name, email}` or an error string |
| POST | `/send-custom-reminders` | `{subject, message, recipients}` | `'all'` → roster emails (broken, see below); `"a@x.com,b@y.com"` → that list |

## Known Limitations & Sharp Edges

Honest notes — all verified against the code and the committed sample data:

- **"Send to all" is broken twice over.** The endpoint calls `readExcelData(filePath, 1)` intending the second sheet (`Contact Info`), but the function's signature is `readExcelData(filePath)` — the index is silently ignored and the **first** sheet is read. Then `.map(item => item.email)` uses lowercase `email` while every committed sheet's header is capital-`E` `Email`, so the mapped list is all `undefined`, and SendGrid rejects each send (logged server-side only). The comma-separated-recipients path works fine.
- **The success message is a lie.** `/send-custom-reminders` responds `'Emails sent successfully'` synchronously; the sends are un-awaited promises whose failures go to `console.error`. The admin's `alert()` says success no matter what.
- **No endpoint has auth.** The `sessionStorage` gate lives entirely in the browser; `curl http://localhost:3000/api/shifts` returns the full schedule **including volunteer email addresses** to anyone who can reach port 3000, and `/send-custom-reminders` will email arbitrary addresses on request.
- **Secrets and paths are edited in source.** The SendGrid key (`''` as committed), the Postgres password (`'2002'`), and an absolute Windows path from a dev machine (`C:\Users\minhk\...`) are all hardcoded constants in [server.js](hackdavis2024/server.js).
- **Plaintext passwords**, stored and compared verbatim — the `insert` and the `where` in [server.js](hackdavis2024/server.js) both take the password as-is. Anything real needs hashing (e.g., bcrypt) plus proper session handling.
- **Error paths can hang requests.** `/login-user` has no `.catch` — a DB error leaves the request with no response at all. `/register-user`'s catch assumes `err.detail` exists and `.includes('already exists')`; any *other* failure (Postgres down, validation error) throws inside the catch and also never responds.
- **The Apps Script re-reminds forever.** `if (diff <= 2)` has no lower bound, so a row whose date is already past matches every single day the trigger runs. And `getDataRange()` pads short rows with empty strings, so a blank cell in the email columns reaches `MailApp.sendEmail('')`, which throws and aborts the rest of that run mid-loop.
- **Frontend leftovers.** `index.html` loads a nonexistent `app.js` (404 on every page view); [home.js](hackdavis2024/public/js/home.js) dereferences `.greeting` and `.logout` elements that exist only in the orphaned [home.css](hackdavis2024/public/css/home.css) design, throwing a `TypeError` on load (the login-redirect gate survives because it's registered first); the logo's `../img/...` path only resolves because browsers clamp `..` at the URL root.
- **Tooling drift.** `package.json` carries three stowaway dependencies from typo'd installs — `cor`, `expres`, and `express.js` — alongside the real `cors` and `express`; `npm start` uses **nodemon** (a dev file-watcher) as the only way to run; and [.vscode/launch.json](hackdavis2024/.vscode/launch.json) debugs against `localhost:8080` while the server listens on 3000.
- **No tests of any kind.** Validation was manual: run the server per [Procedures.md](Procedures.md), click through the pages, watch the console. The Apps Script's only instrumentation is a `Logger.log` of the email count.

## Provenance

A HackDavis 2024 hackathon build — the server directory, npm package, and Postgres database are all named `hackdavis2024`, and the Apps Script sample sheet is dated spring 2024 (`Test Calendar 2024.xlsx`'s schedule rows are mostly spring/summer 2023, plus two rows on 2024-04-28). The product is volunteer scheduling for **Aggie House** (the `aggie_house.png` logo, the `aggiehouseschedules@gmail.com` sender, and the `AggieHouse Reminder` email subject). No authors are named in the code or the original README; the observable traces are the `C:\Users\minhk` dev-machine path in `server.js` and the names filling the sample rosters (Minh Nguyen, Khang Nguyen, Hieu Hoang, Basti). All application code — server, frontend, and Apps Script — is project-authored; the only third-party code is the npm dependency tree (`express`, `knex`, `pg`, `xlsx`, `@sendgrid/mail`, `cors`, `body-parser`, `nodemon`). The original README's setup steps are carried forward above; [Procedures.md](Procedures.md) and [README!.txt](automated_reminder/README!.txt) are the team's original run guides, preserved as-is.

See [SYSTEM-DESIGN.md](SYSTEM-DESIGN.md) for the architecture-level view: the full data-flow diagram, the ideas behind the design, and the numbers that matter.
