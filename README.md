# Day Ledger

Attendance and task accountability for a small team. One Node service, one
Postgres database, no build step.

## What enforces what

The browser cannot be trusted to close a day, so the rules live on the server:

| Rule | Where it's enforced |
|---|---|
| Signing out requires a complete timesheet | `POST /api/logout` returns 409 while a day is open |
| Every open task needs hours and a status | `POST /api/day/close` returns 422 listing what's missing |
| An unfinished day blocks tomorrow's clock-in | `pendingTimesheetFor()` runs at login and at clock-in |
| The clock stops overnight | `cron:close-idle` stamps a provisional clock-out |
| Reminders reach people with no tab open | `cron:remind` sends email and/or Slack |

The disabled button and the red fields in the UI are courtesies. Deleting the
frontend entirely would not weaken a single rule above.

A day auto-closed by the nightly sweep still owes its timesheet — the sweep
sets `close_reason = 'idle_timeout'`, and catch-up keys off missing
`time_entries`, not off the clock-out. So the clock stops but the obligation
doesn't disappear.

## Setup

```bash
cp .env.example .env          # fill in DATABASE_URL and SESSION_SECRET
openssl rand -base64 48       # use this for SESSION_SECRET
npm install
npm run migrate
npm run seed:admin            # reads ADMIN_* from .env, then delete those lines
npm start                     # http://localhost:8080
```

Add the other nine people from the admin bar once you're signed in, or
`POST /api/users`. Everyone gets a temporary password from you; `must_reset`
is set but the reset screen is not built yet (see Known gaps).

## Deploy

Cloud Run is the shortest path if you already have GCP:

```bash
gcloud run deploy dayledger \
  --source . --region asia-south1 --allow-unauthenticated \
  --set-env-vars "NODE_ENV=production,TZ_NAME=Asia/Kolkata,APP_URL=https://..." \
  --set-secrets "DATABASE_URL=dayledger-db:latest,SESSION_SECRET=dayledger-secret:latest"
```

Database: Cloud SQL Postgres (smallest tier) or Neon's free tier. For ten
people either is more than enough.

Then two Cloud Scheduler jobs hitting the container, or plain crontab on a VM:

```
0 9,12,15,18 * * 1-5   cd /app && npm run cron:remind
30 22 * * *            cd /app && npm run cron:close-idle
```

With no SMTP or Slack config the reminder job prints to the console instead of
sending, which is a safe way to watch it for a week before you turn it on.

## Reading the monthly report

`Clock hours` is sign-in to sign-out. `Logged hours` is what people typed into
their timesheets. These will not match, and the gap is the point — a person at
9 clock hours and 4 logged hours is either under-reporting or in meetings all
day, and either way it's worth a conversation. `Days auto-closed` counts days
nobody signed out of properly.

## Known gaps

Things deliberately left for you, in the order I'd do them:

1. **Password reset.** `must_reset` is stored but nothing acts on it. Until
   it's built, you set passwords for people by hand.
2. **Task creation UI.** The admin flow uses `prompt()` — functional, ugly.
   A real form is an hour's work.
3. **Editing a submitted timesheet.** Currently only an admin can override a
   day, and only to close it. People will ask for this in week two.
4. **Rate limiting beyond login.** Fine at ten users, not at fifty.
5. **Backups.** Whatever database you pick, turn on automated backups before
   real data goes in. This is the only item on the list that will hurt you
   permanently if you skip it.

## Not tested end to end

I wrote this without a database to run it against — every file is
syntax-checked, but the SQL and the flows are not exercised. Expect an hour of
small fixes on first run. The two places I'd look first are the `work_date`
comparisons in `server.js` (Postgres returns `DATE` as a JS `Date`, and the
code does `.toISOString().slice(0,10)` to compare) and the CTE date bounds in
`report.js`.

## Shape of the data

See `sql/001_schema.sql`. Five tables: `users`, `tasks`, `work_days` (one row
per person per day — this is attendance), `time_entries` (hours against a task
for a day), and `task_events` (an append-only audit of every status change, so
"who marked this done and when" always has an answer).
