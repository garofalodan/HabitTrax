[README.md](https://github.com/user-attachments/files/32033738/README.md)
# HabitTrax

A single-file, mobile-optimized habit tracker with a dark, colorful UI, passwordless sign-in, an in-app admin invite system, and a Supabase-backed database so your data syncs across devices.

## Features

- **Daily Log** — check off habits for any date, with a date navigator docked above the tab bar
- **Trends** — calendar-aligned lifetime %, an always-visible "this week / this month" status card, a box tracker (contribution-graph style), and a monthly calendar drill-down
- **Habits** — add, edit, reorder (drag), and delete habits
  - 1,776 searchable Lucide icons
  - 20-color palette, auto-assigned with hue-spacing so back-to-back habits don't get similar colors
  - **Repeats:**
    - Daily
    - Weekly — Any Day, X Days a Week (any days), or Specific Days
    - Monthly — Any Day, X Days a Month (any days), or Specific Days (real calendar dates, e.g. the 1st and 15th)
  - **Start Date** — when trend calculations should begin for that habit, editable any time
  - **Holidays/vacations** — exclude date ranges from trends without affecting the Daily Log; add a whole range (From/To) in one action, shown as a single collapsed chip
- **Weeks start on Monday** throughout (calendar, box tracker, weekday picker)
- **Calendar-aligned trends** — weekly/monthly targets reset on real calendar boundaries (every Monday, every 1st of the month), not a rolling lookback window. A historical % reflects only fully-completed periods; a separate "this week/month" card always shows current progress
- **Passwordless sign-in** — email + one-time login code (no passwords), with a "Welcome back" quick sign-in that remembers your email on a given device
- **Invite-only, admin-managed from inside the app** — an admin can send invites directly from the app; the invited account is only created once the recipient actually clicks their link (see Admin Invite System below)
- Works fully offline-capable single file: everything (including the Supabase client library and the icon set) is embedded directly in the HTML — no build step, no external script dependencies

## Tech Stack

- Vanilla JavaScript, HTML, CSS — no framework, no build tools
- [Supabase](https://supabase.com) for auth (email OTP), data storage (Postgres + Row Level Security), and two Edge Functions powering the invite system
- Fonts: Space Grotesk (display), Inter (body), IBM Plex Mono (mono) via Google Fonts

## File Structure

Everything the app itself needs lives in one file: `index.html` (rename `habit-tracker.html` before deploying — see below). It contains:

- The Supabase JS SDK, embedded inline (not loaded from a CDN)
- All app HTML, CSS, and JavaScript
- The full Lucide icon dataset (1,776 icons) embedded as a searchable array

Two small server-side files live outside the HTML, deployed as Supabase Edge Functions (not part of the static site):

- `invite-user-edge-function.ts` — deployed as the `invite-user` function
- `accept-invite-edge-function.ts` — deployed as the `accept-invite` function

## Supabase Setup

### Database

Three tables, all with Row Level Security enabled:

- **`habits`** — name, icon (`emoji` column), color, `frequency` (jsonb), `start_date`, `holidays` (jsonb array of dates), `sort_order`
- **`completions`** — `habit_id`, `date`, unique together
- **`pending_invites`** — `email`, `token`, `created_at`, `expires_at`. No RLS policies at all (fully locked down from the public API) — only Edge Functions using the service role key can touch it. This is what makes the "account isn't created until they click" behavior possible.

Auth uses email OTP (no passwords) with public signups disabled — only invited users can request a sign-in code. The email template is customized to show a numeric code via `{{ .Token }}` instead of a magic link.

Your Supabase project's URL and public (`anon`/`publishable`) key are hardcoded near the top of the `<script>` block:

```js
var SUPABASE_URL = 'https://ijonomuivivguxntwdxa.supabase.co';
var SUPABASE_ANON_KEY = 'sb_publishable_fYniQ6a5XOFpHzii52ZfCg_qCn7QSYD';
```

The admin's email is also hardcoded, purely to control whether the Invite button is shown in the UI (the real security check happens server-side in the Edge Function, not here):

```js
var ADMIN_EMAIL = 'garofalo.dan@gmail.com';
```

If you ever spin up a new Supabase project, update these and re-run the schema.

### Admin Invite System

Instead of Supabase's built-in "invite user" (which creates the account the instant you click Invite), HabitTrax uses a custom flow so **no account exists until the invitee actually clicks their link**:

1. Admin taps the Invite button in the app and enters an email.
2. The `invite-user` Edge Function verifies the caller is really the admin (checked server-side against its own `ADMIN_EMAIL` secret — independent of the client-side one above), writes a row to `pending_invites` with a random token, and emails the invitee a link back to the app (`?invite=<token>`) using Gmail SMTP.
3. The invitee clicks the link. The app detects the `?invite=` parameter and calls the `accept-invite` Edge Function.
4. `accept-invite` validates the token, and **only at this point** creates the real Supabase auth account, then deletes the pending row.
5. The invitee is told to sign in normally — same email + code flow as everyone else.

**Edge Function secrets required** (set at the project level: Edge Functions → Secrets — these apply to all functions, not per-function):

| Secret | Value |
|---|---|
| `ADMIN_EMAIL` | The admin's email — only this account can invite others |
| `APP_URL` | Your GitHub Pages URL, e.g. `https://yourname.github.io/habit-tracker` |
| `GMAIL_USER` | The Gmail address used to send invite emails |
| `GMAIL_APP_PASSWORD` | A 16-character Google App Password (not your normal Gmail password) |

`SUPABASE_URL`, `SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY` are provided automatically by Supabase to every Edge Function — no need to set those yourself.

**Function names matter exactly** — the app calls them by name (`invite-user`, `accept-invite`), all lowercase, one hyphen. If you ever redeploy one, make sure the name is set correctly *at creation time* — renaming an existing function afterward does not reliably change its actual routing endpoint.

## Deploying to GitHub Pages

1. Create a GitHub repository.
2. Rename the file to `index.html` (GitHub Pages serves this as the homepage).
3. Upload it via **Add file → Upload files** on the repo page, then commit.
4. Go to **Settings → Pages**, set Source to "Deploy from a branch," pick `main` and `/ (root)`, and save.
5. After a minute or two, your app will be live at `https://yourusername.github.io/your-repo-name/`.

The first visit to a new URL will require signing in again (the "remembered email" and session are tied to that specific site address), but all habit data itself is stored in Supabase and is identical no matter which URL or device you access it from.

**Note on invite links:** since `APP_URL` (an Edge Function secret) determines where invite links point, invite links always go to your deployed GitHub Pages site — they can never point at a local file on your computer, regardless of where you triggered the invite from.

## Making Future Changes

- **App changes** (anything in `index.html`): re-upload the file to your GitHub repo (Add file → Upload files → same filename → commit). GitHub Pages picks it up automatically within a minute or two.
- **Invite system changes** (either `.ts` file): paste the updated code into the corresponding Edge Function in the Supabase dashboard and redeploy. These are independent of the HTML file — updating one doesn't require updating the other unless a change affects both sides.

## Notes

- No password is ever used — sign-in works purely on a fresh, time-limited code sent to an invited email address.
- All data (habits, completions) is scoped to your Supabase user account, not to any particular device or browser.
- Session/login-related preferences (like the remembered email for quick sign-in) are stored locally per-device and don't affect your actual habit data.
- Trend percentages for weekly/monthly habits reset on real calendar boundaries (Monday, and the 1st of each month) rather than a rolling window — a habit's historical % only reflects fully-completed periods, while a separate "this week/month" card always shows current, in-progress standing.
- The app refetches data from Supabase whenever it regains focus (switching back to the tab, or bringing the phone browser to the foreground), so changes made on one device show up on another without a manual reload — though not instantly while both are sitting open side by side.
