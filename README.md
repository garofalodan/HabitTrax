# HabitTrax

A single-file, mobile-optimized habit tracker with a dark, colorful UI, passwordless sign-in, and a Supabase-backed database so your data syncs across devices.

## Features

- **Daily Log** — check off habits for any date, with a date navigator docked above the tab bar
- **Trends** — per-habit lifetime completion %, a box tracker (contribution-graph style), and a monthly calendar drill-down
- **Habits** — add, edit, reorder (drag), and delete habits
  - 1,776 searchable Lucide icons
  - 20-color palette, auto-assigned with hue-spacing so back-to-back habits don't get similar colors
  - Repeats: Daily, Weekly (Any Day / X Days a Week, any days / Specific Days), or Monthly
- **Weeks start on Monday** throughout (calendar, box tracker, weekday picker)
- **Passwordless sign-in** — email + one-time login code (no passwords), with a "Welcome back" quick sign-in that remembers your email on a given device
- **Invite-only** — only emails you've explicitly added in Supabase can sign in
- Works fully offline-capable single file: everything (including the Supabase client library and the icon set) is embedded directly in the HTML — no build step, no external script dependencies

## Tech Stack

- Vanilla JavaScript, HTML, CSS — no framework, no build tools
- [Supabase](https://supabase.com) for auth (email OTP) and data storage (Postgres + Row Level Security)
- Fonts: Space Grotesk (display), Inter (body), IBM Plex Mono (mono) via Google Fonts

## File Structure

Everything lives in one file: `index.html` (or `habit-tracker.html`, rename before deploying — see below). It contains:

- The Supabase JS SDK, embedded inline (not loaded from a CDN)
- All app HTML, CSS, and JavaScript
- The full Lucide icon dataset (1,776 icons) embedded as a searchable array

## Supabase Setup

The app expects two tables (`habits` and `completions`) with Row Level Security tied to `auth.uid()`, and email-based OTP auth with public signups disabled (invite-only).

Your Supabase project's URL and public (`anon`/`publishable`) key are hardcoded near the top of the `<script>` block:

```js
var SUPABASE_URL = 'https://ijonomuivivguxntwdxa.supabase.co';
var SUPABASE_ANON_KEY = 'sb_publishable_fYniQ6a5XOFpHzii52ZfCg_qCn7QSYD';
```

If you ever spin up a new Supabase project, update these two lines and re-run the schema (habits + completions tables, RLS policies, and the email OTP template configured to include `{{ .Token }}`).

**Inviting a new user:** add their email in the Supabase dashboard under Authentication → Users → Invite. Only invited emails can request a sign-in code.

## Deploying to GitHub Pages

1. Create a GitHub repository.
2. Rename the file to `index.html` (GitHub Pages serves this as the homepage).
3. Upload it via **Add file → Upload files** on the repo page, then commit.
4. Go to **Settings → Pages**, set Source to "Deploy from a branch," pick `main` and `/ (root)`, and save.
5. After a minute or two, your app will be live at `https://yourusername.github.io/your-repo-name/`.

The first visit to a new URL will require signing in again (the "remembered email" and session are tied to that specific site address), but all habit data itself is stored in Supabase and is identical no matter which URL or device you access it from.

## Making Future Changes

Since everything is one file, updating the live app just means re-uploading a new version of `index.html` to the same GitHub repo (Add file → Upload files → same filename → commit). GitHub Pages picks up the change automatically within a minute or two.

## Notes

- No password is ever used — sign-in works purely on a fresh, time-limited code sent to an invited email address.
- All data (habits, completions) is scoped to your Supabase user account, not to any particular device or browser.
- Session/login-related preferences (like the remembered email for quick sign-in) are stored locally per-device and don't affect your actual habit data.
