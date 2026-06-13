# Rounds App — Development Notes
*Last updated: June 12, 2026*

---

## Project Overview
A mobile-first building rounds web app for 343 Sansome St. consisting of:
- **Mobile field app** (`index.html`) — engineers use on phones to log readings
- **Admin panel** (`admin.html`) — Scott uses on desktop to configure templates and log points
- **Backend** — Supabase (fully migrated from Google Apps Script)
- **Hosting** — GitHub Pages at `https://smmetro.github.io/343-rounds/`
- **Edge Function** — `manage-users` deployed on Supabase for secure user management

**Engineer team:** Scott Morris — `smorris@metroservices.com` (admin) + Jacky Cheung — `jacky@metroservices.com`
**Alert email:** smorris@metroservices.com

---

## Architecture Decisions

### App Name
- Settled on **"Rounds"** — universal, works for daily rounds, chiller rounds, meter readings, etc.
- Designed to be shareable with other engineers industry-wide (not 343-specific)

### Hosting
- **GitHub Pages** — free, connected to `smmetro/343-rounds` repo
- Files: `index.html` (mobile), `admin.html` (admin), `.nojekyll` (prevents Jekyll build timeouts)
- Attempted Cloudflare Pages — ran into project name conflict ("rounds" taken globally), Workers URL instead of Pages URL, abandoned

### Config Storage — EVOLUTION
1. **localStorage only** — original approach. Config saved on each device independently. Problem: no sync between devices.
2. **Google Apps Script webhook** — Save & Publish writes config to Sheet, mobile app fetches on load. Problem: CORS restrictions blocked all requests from iOS.
3. **Supabase (current)** — proper backend with CORS support. Solves sync and cross-origin issues permanently.

### Authentication — EVOLUTION
1. **Custom username/password** — original approach. Passwords hashed in JS (`hashPw`), stored in `users` table as `pw_hash`. Session stored in localStorage (`lc_session`). Problem: anon key exposed in HTML gave anyone full database access.
2. **Supabase Auth (current)** — engineers log in with email + password via `db.auth.signInWithPassword()`. Supabase issues a short-lived JWT. All DB queries require a valid authenticated session. Anon key is now effectively useless.

### PWA / Home Screen Icon
- Added PWA manifest, apple-touch-icon, and service worker early in project
- **Service worker caused major problems** — aggressively cached old versions, prevented updates from reaching phones
- **Solution:** Removed service worker entirely, replaced with unregister code
- Home screen icon still works without service worker — just loses offline capability
- Icon design: dark background (#0f1117), blue circle (#4f8ef7), white checkmark, no text

### Google Apps Script — ABANDONED
- **Fatal problem:** Google Apps Script intentionally blocks cross-origin requests from mobile web apps
- Root cause: Apps Script was never designed to be called from mobile web apps
- Replaced entirely by Supabase

### localStorage Pitfalls
- localStorage is scoped per origin AND per browser context
  - Safari browser ≠ Safari PWA (home screen) — completely separate storage
  - Local file (`file://`) ≠ GitHub Pages (`https://`) — completely separate storage
- No longer used for auth sessions (Supabase manages sessions internally)
- Still used for: round history cache, offline fallback, config cache

---

## Supabase Setup

### Project
- **Project ID:** `hczgckuggechzybulded`
- **URL:** `https://hczgckuggechzybulded.supabase.co`
- **Anon key:** `sb_publishable_vvyTcuHXfJN1hydx8OgJyQ_y1Q7FeIp` (safe in HTML — all tables require authenticated JWT)

### Tables
| Table | Purpose |
|-------|---------|
| `app_config` | Stores app configuration JSON (one row, id = 'main') |
| `rounds` | Completed round submissions |
| `users` | Engineer profiles — display name, email, admin status, assigned rounds |

### users table schema
```
username        text NOT NULL       — display name shown in app
email           text                — login email for Supabase Auth
pw_hash         text                — DEPRECATED, no longer used
is_admin        boolean             — admin panel access
assigned_rounds text                — 'all' or array of template IDs
force_reset     boolean             — DEPRECATED, no longer used
created_at      timestamptz
auth_uid        uuid UNIQUE         — links to Supabase Auth user (auth.users.id)
```

### Row Level Security (RLS)
All three tables have RLS enabled. Policies:

**users:**
- Authenticated users can SELECT their own row (`auth_uid = auth.uid()`)
- Admins can SELECT all rows
- Admins can UPDATE any row (for makeAdmin, removeAdmin, round assignments)
- INSERT/DELETE go through Edge Function (service_role bypasses RLS)

**rounds:**
- Authenticated users can SELECT all rounds (for previous reading display)
- Authenticated users can INSERT (to submit a round)

**app_config:**
- Authenticated users can SELECT (to load templates on app start)
- Admins can INSERT and UPDATE (Save & Publish)

### is_admin() helper function
```sql
CREATE OR REPLACE FUNCTION public.is_admin()
RETURNS boolean LANGUAGE sql SECURITY DEFINER STABLE AS $$
  SELECT COALESCE(
    (SELECT is_admin FROM public.users WHERE auth_uid = auth.uid()),
    false
  );
$$;
```
`SECURITY DEFINER` is required to prevent infinite recursion in RLS policies.

### Google Sheets Sync
- Sheet name: **"343 Sansome Daily Rounds"**
- Apps Script file: `SheetsSync.gs` (v7)
- Sync method: Supabase database webhook fires on `rounds` INSERT
- Two-row merged section headers in sheet ✅
- Keep-alive ping every 12 hours ✅

---

## Edge Function — manage-users

### Purpose
Handles user management operations that require the `service_role` key (which must never be in any HTML file). Called only from `admin.html` by authenticated admins.

### URL
`https://hczgckuggechzybulded.supabase.co/functions/v1/manage-users`

### Actions
| Action | What it does |
|--------|-------------|
| `create` | `inviteUserByEmail` → creates Supabase Auth account + sends invite email → inserts `users` row |
| `delete` | Deletes `users` row + deletes Supabase Auth account |
| `reset-password` | Sends password recovery email via `generateLink` |
| `update` | Updates fields in `users` table |

### Security
- Verifies caller JWT via anon key client
- Checks `users` table for `is_admin = true`
- Only then uses service_role client for admin operations
- CORS restricted to `https://smmetro.github.io`

### Calling from admin.html
```javascript
const EDGE_FN_URL = 'https://hczgckuggechzybulded.supabase.co/functions/v1/manage-users';

async function callEdgeFn(action, params) {
  const { data: { session } } = await db.auth.getSession();
  const res = await fetch(EDGE_FN_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + session.access_token },
    body: JSON.stringify({ action, ...params })
  });
  const result = await res.json();
  if (!result.success) throw new Error(result.error || 'Unknown error');
  return result;
}
```

---

## Adding New Engineers

### For brand new engineers (no existing users row)
1. Admin panel → Users → Create User → enter display name, email, role
2. Edge Function creates Auth account + sends invite email + inserts users row
3. Engineer clicks invite link, sets password, logs into mobile app

### For engineers with existing users rows (migration scenario)
1. Supabase dashboard → Authentication → Users → Add user → enter email + set temporary password
2. Copy UUID from the new Auth user row
3. SQL Editor: `UPDATE public.users SET auth_uid = 'UUID' WHERE username = 'Name';`
4. Give engineer their temporary password

### Email rate limits
Supabase free tier limits outgoing emails to ~2/hour. If invite emails don't arrive:
- Check spam folder first
- Dashboard → Authentication → Users → find user → copy recovery link directly
- Or set password directly in the dashboard (Add user instead of Invite)
- Long-term fix: configure a custom SMTP provider (SendGrid or Resend both have free tiers) under Authentication → Settings → SMTP

---

## Features Built

### Mobile App (`index.html`)
- Login screen — email + password via Supabase Auth
- Password set screen — triggered by invite/recovery email link (handles `PASSWORD_RECOVERY` auth event)
- Round type selection screen (pulls from Supabase config)
- Log point entry with:
  - Numeric input with units
  - Status/button selection with color coding (OK/warn/alarm)
  - Warning and alert thresholds with acknowledgment required
  - Conditional logic (log points that only appear based on previous answers)
  - Notes (text) per log point
  - Photo capture per log point
- Jump-to navigator (list all log points, jump to any)
- Previous reading display
- Round complete screen with sync status
- Saves to localStorage as fallback when offline
- Emergency reset button (faint, bottom of login screen)
- PWA: home screen icon, full-screen mode, no browser chrome
- **Scroll-to-top on every log point navigation** (fixed June 2026)

### Admin Panel (`admin.html`)
- Admin-only login via Supabase Auth (email + password)
- Settings: building name, alert emails
- Template management: create/edit/delete/reorder round templates
- Log point management per template:
  - Name, description, section, type (numeric/status)
  - Units, warn/alert thresholds
  - Conditional logic configuration
  - Status options with styling
  - Duplicate log point button
- User management: create engineers (via Edge Function), assign rounds, send password reset, delete
- Drag-and-drop reordering of log points and sections
- Save & Publish: writes config to Supabase, syncs to all devices

---

## Current State (as of June 12, 2026)
- ✅ Mobile app login via Supabase Auth
- ✅ Admin panel login via Supabase Auth
- ✅ Config syncs to all devices via Supabase
- ✅ Round submissions sync to Supabase
- ✅ Google Sheets auto-updates via webhook (SheetsSync.gs v7)
- ✅ Two-row merged section headers in Sheet
- ✅ Keep-alive ping every 12 hours
- ✅ RLS fully locked down — anon key has zero table access
- ✅ User management via Edge Function (service_role key never in HTML)
- ✅ Back to Rounds button fixed (async/await bug in resetRound)
- ✅ Scroll-to-top fixed on log point navigation
- ✅ Emoji picker working in admin template editor
- ⏳ **Photos/notes sync not yet implemented**
- ⏳ **Custom SMTP provider not yet configured** (Supabase free tier email rate limit)

---

## Known Issues / Watch Out For
- **iOS 26 Safari:** Settings > Safari not accessible directly — must go Settings > Apps > Safari
- **Service workers on iOS:** Extremely persistent cache, very difficult to clear. Avoid using aggressive caching.
- **GitHub Pages:** `.nojekyll` file required to prevent Jekyll build timeouts on large HTML files
- **Supabase email rate limit:** Free tier allows ~2 emails/hour. Use "Add user" + manual password in dashboard as workaround. Fix properly by configuring custom SMTP.
- **Supabase public schema Data API change:** Effective October 30, 2026 for existing projects — any NEW tables created after that date will require explicit `GRANT` before they're accessible via PostgREST/supabase-js. Existing tables (app_config, rounds, users) are grandfathered. When adding new tables, run: `GRANT SELECT, INSERT, UPDATE, DELETE ON public.new_table TO anon, authenticated;`
- **Google Apps Script CORS:** Cannot be fixed — do not attempt to use Apps Script for mobile API calls
- **Cloudflare Pages:** Project name "rounds" is globally reserved — use `smmetro-rounds` or `343-rounds`
- **resetRound() bug (fixed):** Was calling `getUsers()` without `await` — returned a Promise instead of array, `.find()` always returned undefined, sent user to login instead of rounds. Fixed June 2026.
