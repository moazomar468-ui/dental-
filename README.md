# Dr. Mohamed Haider Dental Clinic — Management & Online Booking System

A complete full-stack clinic system: public website, patient portal (registration,
booking, records, messaging) and a private doctor dashboard, backed by a real
MySQL database with secure credential authentication.

## Stack

- **Frontend** — React 19 + TypeScript + Vite + Tailwind CSS
- **Backend** — Hono + tRPC 11 (type-safe APIs, Zod-validated)
- **Database** — MySQL via Drizzle ORM (`db/schema.ts`)
- **Auth** — email + password, scrypt hashing, server-side sessions (httpOnly cookie)
- **Files** — patient documents in platform object storage
- **Email** — Nodemailer with SMTP env config + full outbox log in the database

## Run

```bash
npm install
npm run db:push     # create tables
npx tsx db/seed.ts  # doctor account, services, working hours
npm run dev         # http://localhost:3000
```

Production: `npm run build && npm start`.

## Accounts after seeding

| Role | Email | Password |
|---|---|---|
| Doctor | `doctor@haiderdental.com` | `Haider@Clinic2026` |

Change these via `DOCTOR_EMAIL` / `DOCTOR_PASSWORD` env before the first seed,
and change the password after first sign-in (Profile → Password). Patients
self-register on the site.

## Email configuration (SMTP)

Every notification (confirmation, reminder, cancellation, thank-you, follow-up)
is always composed and stored in the **outbox** (doctor dashboard →
Notifications). To deliver them as real emails, set these environment variables
on the server (any provider works — Gmail, Outlook, SendGrid, Resend, Amazon SES):

```bash
SMTP_HOST=smtp.resend.com        # or smtp.gmail.com, smtp.sendgrid.net, …
SMTP_PORT=587                    # 465 with SMTP_SECURE=true
SMTP_SECURE=false
SMTP_USER=your-smtp-username
SMTP_PASS=your-smtp-password
SMTP_FROM="Dr. Mohamed Haider Dental Clinic <care@your-domain.com>"
```

Restart the server after setting them. The dashboard shows “Email service
connected” once `SMTP_HOST` is present, and outbox rows flip from `logged`
to `sent`.

## Automatic daily reminders

Reminders for tomorrow’s confirmed appointments can be triggered two ways:

1. **Manually** — doctor dashboard → Notifications → “Send tomorrow’s reminders”.
2. **Automatically** — point any scheduler (cron-job.org, server crontab,
   GitHub Actions) at the built-in endpoint, once per day:

   ```bash
   curl -X POST https://<your-site>/api/cron/send-reminders \
        -H "x-cron-secret: <APP_SECRET from .env>"
   ```

   Reminders are sent at most once per appointment (deduplicated via the outbox).

## Security notes

- Passwords are scrypt-hashed; sessions live server-side (hashed tokens) in an
  httpOnly cookie — no tokens or passwords ever reach frontend storage.
- Role guards: `patientQuery` (signed-in) and `doctorQuery` (clinic staff only)
  protect every tRPC procedure; medical records and documents are additionally
  scoped to the owning patient.
- Double booking is impossible: the `activeSlotKey` unique index rejects
  concurrent bookings of the same slot at the database level.
- Patient documents are stored in object storage, not the database, and are
  served through short-lived presigned URLs with ownership checks.
- All inputs are Zod-validated server-side; secrets stay in `.env` (server only).
