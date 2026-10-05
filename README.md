# Technology as Seva

A real, multi-page Seva platform inspired respectfully by Swami Vivekananda's teachings.
**React + Vite** front end · **Supabase** (Postgres, Auth with e-mail OTP, Realtime, Storage, Edge Functions) back end · **Claude** for the AI layer.

Nothing in the UI is simulated: sign-up sends a real OTP e-mail, chat/notifications/status are pushed live
from the database, uploads go to private cloud storage, and AI answers come from a server-side function
(or the app says clearly that AI is not configured).

---------------------------------------------------------------------------

## 1. What you must provide (keys & accounts)

| # | What | Where it goes | Secret? | Required? |
|---|------|---------------|---------|-----------|
| 1 | **Supabase project** (free tier is fine) → `Project URL` + `anon public key` | `.env` → `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` | No — these two are public by design | **Yes** |
| 2 | **SMTP sender** for OTP e-mails (Resend, Brevo, SendGrid, Gmail app-password…) | Supabase → Authentication → SMTP Settings | Yes — stays inside Supabase | **Yes** for real use* |
| 3 | **Anthropic API key** (console.anthropic.com) | Supabase secret: `supabase secrets set ANTHROPIC_API_KEY=...` | **Yes — never in `.env`/frontend** | Optional (AI features show "not set up" without it) |

\* Supabase's built-in mail sender is heavily rate-limited (only a few e-mails/hour, and only to your team's addresses). It is fine for a first test, not for a community.

**Do not paste secret keys (service-role key, Anthropic key, SMTP password) into any chat.** Put them only in Supabase's dashboard / `supabase secrets`.

---------------------------------------------------------------------------

## 2. Setup (about 15 minutes)

### a) Database
1. Create a project at supabase.com.
2. SQL Editor → run, **in order**, the three files in `supabase/migrations/`:
   `001_schema.sql`, `002_seed_philosophy.sql`, `003_public_stats.sql`.
   (This creates every table, security rule, workflow function, the private `resources` storage bucket, and turns on Realtime.)

### b) Real e-mail OTP
Supabase Dashboard → **Authentication**:
1. *Providers → Email*: enable it and keep **Confirm email = ON** (the app refuses to continue if it is off).
2. *SMTP Settings*: enable custom SMTP with your provider's credentials and a sender address.
3. *Email Templates* → **Confirm signup**: replace the body so it shows the 6-digit code:
   ```html
   <h2>Welcome to Technology as Seva</h2>
   <p>Your verification code is:</p>
   <h1 style="letter-spacing:6px">{{ .Token }}</h1>
   <p>It expires soon. If you didn't sign up, ignore this e-mail.</p>
   ```
4. *Email Templates* → **Reset password**: same idea with `{{ .Token }}` (the app uses a code, not a link).
5. *Settings → Auth*: set OTP length to **6** and a sensible expiry (e.g. 3600 s).

### c) Run the site
```bash
cp .env.example .env     # fill VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY
npm install
npm run dev              # http://localhost:5173
```

### d) Make your first admin
Register normally (verify the e-mail), then in the SQL editor:
```sql
update public.profiles set role = 'admin'
where id = (select id from auth.users where email = 'you@example.com');
```
Admins cannot be created through the website (by design).

### e) AI Seva Engine (optional but recommended)
```bash
npm i -g supabase                      # or: npx supabase ...
supabase login
supabase link --project-ref YOUR_PROJECT_REF
supabase secrets set ANTHROPIC_API_KEY=sk-ant-...        # server-side only
# optional: supabase secrets set ANTHROPIC_MODEL=claude-sonnet-5-5  ALLOWED_ORIGIN=https://your-domain
supabase functions deploy seva-ai
```
Without this step everything still works; matching uses the transparent database score and AI panels say they are not configured.

### f) Optional: feedback reminders after 24 h
Enable the `pg_cron` extension, then:
```sql
select cron.schedule('seva-feedback-reminders', '0 * * * *', $$select public.send_feedback_reminders()$$);
```

### g) Deploy the front end
`npm run build` → upload `dist/` to Netlify / Vercel / Cloudflare Pages (set the two `VITE_` variables there and add a rewrite of all paths to `/index.html`). Add your site URL under Supabase → Authentication → URL Configuration.

---------------------------------------------------------------------------

## 3. How the requirements are met

* **Real auth** – Supabase Auth: register → e-mailed 6-digit OTP → `verifyOtp` → active. Login, logout, forgot-password (by code), session persistence, protected + role-based routes.
  Unverified accounts cannot get a session, *and* every database policy/RPC independently checks `email_confirmed_at`.
* **Roles** – chosen at registration (`seeker` / `volunteer`); `admin` can only be granted in SQL. A trigger blocks users from changing their own role.
* **Workflow integrity** – accept/start/complete/confirm/cancel/dispute are Postgres functions that enforce who may act and in what order. Clients cannot write those tables directly.
* **Realtime sync** – one `seva_connections` row is the single truth; Supabase Realtime pushes changes to both browsers (status, chat, notifications, availability).
* **Privacy** – chat visible only to the two participants; admins cannot read it except via an *audited* function for a disputed/reported Seva; no e-mail/phone ever exposed (public volunteer data comes from `volunteer_cards()` which returns safe fields only); private storage bucket with signed URLs; 10 MB / PDF-PNG-JPEG-TXT-DOCX limits in bucket + client.
* **Philosophy** – 10 teachings in `src/data/philosophy.js` (seeded to the DB; admin-editable). Quotations are short, widely-published passages from the public-domain *Complete Works*; platform-written explanations are labelled separately. **Before launch, confirm exact volume/page citations against the Ramakrishna Mission edition.**
* **No leaderboards / no seeker ratings** – enforced in code (`submit_feedback` nulls any volunteer→seeker rating).

## 4. Tests

`npm run test:db` loads the real migrations into an in-memory Postgres (PGlite with Supabase-style stubs) and runs 88 checks covering the whole flow and the security rules
(unverified access, role escalation, chat privacy, mutual confirmation, record creation, disputes, admin audit, suspended users, etc.).

## 5. Honest notes / what to review before a public launch
* Legal pages are starter text — have a lawyer review them.
* The hero photo is loaded from Wikimedia Commons (public domain); if it cannot load, an original lotus placeholder is shown.
* "Volunteer verified" = verified e-mail. Real-world identity checks (ID, references) are *not* included; the admin "Community reviewed" badge is the hook for adding them.
* Add rate limiting / CAPTCHA (Supabase Auth settings) before opening registration to the public.
* Not a substitute for professional help; the Terms say so, and the platform is for guidance and information.
