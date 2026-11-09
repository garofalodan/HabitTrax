# HabitTrax

A single-file, mobile-first habit tracker with a dark, colorful UI, email + password sign-in, an in-app admin invite system, and a Supabase database so your data syncs across devices. Built for the iPhone and installable to the Home Screen like an app.

## Features

- **Daily Log**: check off habits for any date. Every habit shows its type under the name (Daily, Weekly, Monthly, and so on). A date navigator is docked above the tab bar.
- **Trends**: a list of habits with lifetime completion %. Tap a habit for its detail page:
  - the habit's type, shown under its name
  - an always-visible "this week / this month" status card
  - a box tracker (contribution-graph style) and a monthly calendar that agree with each other
  - previous/next arrows to move between habits
  - a legend: **Completed**, **Not Completed**, and **Exception**
- **Habits**: add, edit, reorder (drag), and delete habits.
  - 1,776 searchable Lucide icons
  - 20-color palette, auto-assigned with hue spacing so back-to-back habits don't look alike
  - **Repeats:**
    - Daily
    - Weekly: Any Day, X Days a Week (any days), or Specific Days
    - Monthly: Any Day, X Days a Month (any days), or Specific Days (real calendar dates, such as the 1st and 15th)
  - **Start Date**: when trend calculations begin for that habit. Editable any time.
  - **Exceptions**: dates you don't need to log a habit, such as vacation or illness. Add a whole range (From/To) in one action. Exceptions are excluded from trends and streaks without affecting the Daily Log, and you can still check the habit off on those days.
- **Calendar-aligned trends**: weekly and monthly targets reset on real calendar boundaries (every Monday, every 1st of the month), not a rolling window. The historical % reflects only fully completed periods, and the "this week/month" card always shows current progress.
- **Weeks start on Monday** throughout.
- **Sign-in**: email + password, with a one-time email code for first-time sign-in and forgotten passwords. "Stay logged in" is on by default. The last-used email is remembered on each device.
- **Invite-only**: an admin invites people from inside the app. Their account is only created when they click the link.
- **Home Screen app**: opens full-screen on iPhone with no browser bars when added from Safari (see below).
- Everything (Supabase client, icon set, app code) is embedded in one HTML file. No build step.

## Tech stack

- Vanilla JavaScript, HTML, CSS.
- [Supabase](https://supabase.com): Auth, Postgres with Row Level Security, Edge Functions.
- Gmail SMTP for sign-in code emails and invite emails.
- Fonts via Google Fonts: Space Grotesk, Inter, IBM Plex Mono.

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole app. This is what you deploy to GitHub Pages. |
| `invite-user-edge-function.ts` | Source for the `invite-user` Edge Function. |
| `accept-invite-edge-function.ts` | Source for the `accept-invite` Edge Function. |
| `pending-invites-schema.sql` | SQL for the invites table (also included below). |
| `habittrax-icon-180.png`, `habittrax-icon-512.png` | App icons (already embedded in `index.html`). |

---

## Install on iPhone (full-screen, no browser bars)

1. Open your deployed site in **Safari** (not Chrome).
2. Tap **Share**, then **Add to Home Screen**.
3. Open HabitTrax from the new icon.

Notes:
- The Home Screen app keeps its own login, separate from Safari. Sign in once on first launch.
- If you added an icon before the full-screen settings were in `index.html`, delete it and add it again.
- Quick alternative without installing: in Safari, tap **aA** in the address bar, then **Hide Toolbar**. This is temporary.
- After you update `index.html` on GitHub, close and reopen the Home Screen app to pick up the new version. This can take a few minutes.

---

## Fresh setup (new or wiped Supabase project)

Follow these in order. The order matters: the admin user must exist **before** you turn off public signups, because the app never creates users by itself.

### 1. Create the project

Create a Supabase project, then copy two values from **Project Settings → API**:

- **Project URL**, e.g. `https://xxxx.supabase.co`
- **Publishable (anon) key**, e.g. `sb_publishable_...` (safe to put in the HTML)

Never put the `service_role` / secret key in the HTML. It only belongs in Edge Functions, which Supabase provides to them automatically.

### 2. Run the database SQL

Open **SQL Editor → New query**, paste, and run:

```sql
-- Habits
create table habits (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  emoji text not null,              -- stores the Lucide icon name
  color text not null,
  frequency jsonb not null default '{"type":"daily"}',
  sort_order integer not null default 0,
  created_at timestamptz not null default now(),
  start_date date,
  holidays jsonb default '[]'::jsonb  -- stores a habit's Exception dates (the app labels these "Exceptions")
);

-- Completions (one row per habit per day)
create table completions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  habit_id uuid not null references habits(id) on delete cascade,
  date date not null,
  created_at timestamptz not null default now(),
  unique (habit_id, date)
);

-- Row Level Security: users only see and change their own rows
alter table habits enable row level security;
alter table completions enable row level security;

create policy "Users manage own habits" on habits
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "Users manage own completions" on completions
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- Pending invites: RLS on with NO policies, so only Edge Functions
-- (service role) can read or write it.
create table pending_invites (
  id uuid primary key default gen_random_uuid(),
  email text not null,
  token text not null unique,
  created_at timestamptz not null default now(),
  expires_at timestamptz not null default (now() + interval '7 days')
);

alter table pending_invites enable row level security;
```

The `holidays` column is where Exception dates are stored. The name is kept for compatibility, so existing data keeps working. Don't rename it unless you also change `index.html`.

### 3. Create the admin user (do this before disabling signups)

**Authentication → Users → Add user → Create new user.** Enter your admin email, leave the password blank or set a temporary one, and make sure **Auto Confirm User** is checked.

On first sign-in you'll use the email code flow and then choose your real password.

### 4. Configure Auth

1. **Authentication → Sign In / Providers**: Email provider enabled. Turn **off** "Allow new users to sign up" (this makes the app invite-only; existing users can still sign in).
2. **Authentication → Sign In / Providers → Email**: set **Email OTP length to 6**.
3. **Authentication → Emails → Templates → Magic Link**: replace the body so it shows the code instead of a link:

   ```html
   <h2>Your HabitTrax sign-in code</h2>
   <p>Enter this code in the app:</p>
   <h1>{{ .Token }}</h1>
   <p>It expires shortly. If you didn't request it, ignore this email.</p>
   ```

4. **Authentication → Emails → SMTP Settings**: enable **Custom SMTP** so codes can be sent to more than the team's own addresses.
   - Host: `smtp.gmail.com`
   - Port: `465`
   - Username: your Gmail address
   - Password: a 16-character Google App Password
   - Sender email/name: your Gmail address / `HabitTrax`

### 5. Create the Google App Password

Google Account → Security → turn on 2-Step Verification → search "App passwords" → create one named HabitTrax. You get 16 characters. Enter them **without spaces**. Use this for both the SMTP password above and the `GMAIL_APP_PASSWORD` secret below. Do not paste it into chat or commit it anywhere.

### 6. Edge Function secrets

**Edge Functions → Secrets** (project-level, applies to every function):

| Secret | Value |
|---|---|
| `ADMIN_EMAIL` | Admin email. Only this account can send invites. |
| `APP_URL` | Your deployed site, e.g. `https://yourname.github.io/habit-tracker` (no trailing slash) |
| `GMAIL_USER` | Gmail address that sends invite emails |
| `GMAIL_APP_PASSWORD` | The 16-character app password, no spaces |

`SUPABASE_URL`, `SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY` are provided automatically.

### 7. Deploy the Edge Functions

**Edge Functions → Deploy a new function → Via Editor.** Create exactly two functions:

| Function name | Paste in code from |
|---|---|
| `invite-user` | `invite-user-edge-function.ts` |
| `accept-invite` | `accept-invite-edge-function.ts` |

Names must be exact: lowercase, one hyphen. Set the name when you **create** the function. Renaming one afterward in the dashboard does not change its routing slug, and the app will get a 404. If that happens, delete it and create it fresh with the right name.

### 8. Point the app at your project

Near the top of the `<script>` in `index.html`, update these three values:

```js
var SUPABASE_URL = 'https://YOUR-PROJECT.supabase.co';
var SUPABASE_ANON_KEY = 'sb_publishable_...';
var ADMIN_EMAIL = 'you@example.com';
```

`ADMIN_EMAIL` here only controls whether the Invite button and your name in the greeting are shown. The real admin check happens server-side against the `ADMIN_EMAIL` secret.

### 9. Deploy to GitHub Pages

1. Create a GitHub repo.
2. Upload `index.html` to the repo root (**Add file → Upload files**) and commit.
3. **Settings → Pages**: Source = "Deploy from a branch", branch `main`, folder `/ (root)`. Save.
4. After a minute or two it's live at `https://yourname.github.io/your-repo-name/`. Make sure this matches `APP_URL`.

### 10. First sign-in

Open the site, tap **First time? Use an email code and set your password**, enter the admin email, enter the 6-digit code, then choose a password. You're in, and the Invite button is visible.

---

## Sign-in flow

- **Returning users:** email + password. The "Stay logged in" checkbox (on by default) keeps you signed in across browser restarts. Unchecked, the session ends when the browser or tab closes.
- **First time:** request an email code, enter it, then set a password. A password is required; there is no skip.
- **Forgot password:** use the same email-code option. After the code, you set a new password, which overwrites the old one.
- The last-used email is remembered on each device for a quick "welcome back" sign-in.

## Invite system

1. Admin taps the Invite button and enters an email.
2. `invite-user` checks that the caller is the admin (server-side), stores a row in `pending_invites` with a random token, and emails a link (`APP_URL/?invite=<token>`) via Gmail SMTP. Links expire after 7 days.
3. The invitee opens the link. The app calls `accept-invite`, which validates the token and **only then** creates the Supabase auth account, then deletes the pending row.
4. The invitee continues through the email-code flow and sets a password.

## Updating later

- **App changes:** re-upload `index.html` to the repo (same filename). Pages updates in a minute or two.
- **Edge Function changes:** paste the updated code into the function in the Supabase dashboard and redeploy.
- On iPhone, you may need to close and reopen the Home Screen app to pick up a new version.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| "Failed to send a request to the Edge Function" / 404 | Function name is wrong. Recreate it with the exact name. |
| "Edge Function returned a non-2xx status code" | A secret is missing, or the caller isn't the admin. Check the four secrets. |
| Email error `InvalidContentType` | SMTP is on port 587. Use port 465 with TLS in the function and in Supabase SMTP settings. |
| Email shows a link instead of a code | The Magic Link template doesn't contain `{{ .Token }}`. |
| Code is 8 digits | Set Email OTP length to 6. |
| "Could not send a code" | The email isn't an existing user (signups are off). Invite them, or create the admin user first. |
| Invite button missing | You aren't signed in as the `ADMIN_EMAIL` set in `index.html`. |
| Invite link goes to the wrong place | `APP_URL` secret is wrong. |
| Browser bars still show on iPhone | Open it from the Home Screen icon (added from Safari), not from a Safari or Chrome tab. Delete and re-add an older icon. |
| Supabase project paused | Free projects pause after 7 days of low activity. Open the dashboard and click **Resume project**. Data is kept. |

## Notes

- All habit data is stored per user in Supabase and is identical on every device. Only preferences like the remembered email and the stay-logged-in choice are stored locally.
- The app refetches whenever it regains focus, so changes on one device appear on another after switching back to it.
- The publishable key is safe to expose. The service role key and Gmail app password must never appear in `index.html` or the repo.
