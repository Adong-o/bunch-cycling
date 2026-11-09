# Bunch

A cycling tracker — Strava-shaped, cycling-only, $4.99/mo Premium. Two pieces in this folder:

```
mobile/      Expo React Native app (iOS + Android) — the product
             mobile/supabase/schema.sql — the entire backend in ONE file: tables, RLS policies,
             triggers, views, storage buckets, seed data and a self-check
website/     Public marketing site + the admin console (day-to-day: users, challenges, badges,
             moderation), nested under /admin — one Vite app, see website/README.md
```

Both apps already point at a live Supabase project (`.env` in each folder is filled in). If that is
the project you want to keep using, run the new `schema.sql` once in its SQL Editor (it upgrades
whatever you have and never drops data), then go to **Build the Android APK**. If you're starting a
fresh project, do the three setup steps first.

## What changed in 1.5

**The app stopped closing on you.** The cause was a class of problems rather than one bug, so the fix
is layered:

- `src/lib/crashGuard.ts` is imported first in `App.tsx` and installs a global handler. In a release
  build React Native closes the app on *any* uncaught JavaScript error (a throw in a button handler,
  a timer, a native callback). Now the error is journaled and the rider gets a calm screen with
  **Restart Bunch** (`components/CrashGate.tsx`, which also catches render errors). The JS runtime is
  healthy after such an error, so a ride being recorded keeps recording underneath.
- A build made **without its Supabase keys** used to throw on launch and vanish. It now opens on a
  screen that says exactly which two variables are missing.
- `src/lib/errorLog.ts` keeps the last 40 errors on the phone, and **Settings → Diagnostics** shows
  them next to a database check (it compares `bunch_schema_version()` with the version the app
  expects, so "I forgot to run the schema" is one glance, not a hunt). Errors are uploaded to the
  `app_errors` table when there is a connection, so "it closed on me" now leaves evidence.
- Low-memory phones: long lists render a small window of rows (`lib/listPerf.ts`), and unbounded
  fetches (heatmap tracks, ride history) are capped or paged.

**Navigation icons were redrawn** (`components/icons/HeroIcons.tsx`, `navigation/AppTabBar.tsx`):
one consistent stroke weight, a filled state for the selected tab, a small spring when you switch.

**Rider analytics and graphs** — the Stats screen (Profile → Stats) is rebuilt around your own rides:

- A range switch (4 weeks / 12 weeks / 1 year / All) and a metric switch (Distance / Time / Climbing)
  drive one bar chart. Tap or drag across it to read a day, week or month; **Show data** turns the
  chart into a table.
- The headline number is compared with the *previous stretch of the same length* — and only when you
  were riding back then, so a rider with three weeks of history never sees a "+667%".
- Tiles for rides, days ridden, average ride and average climb; an average-speed trend that leaves
  e-bike rides out when you also have acoustic ones; **When you ride** (weekday and time of day);
  distance by bike; ride-length bands; a consistency calendar with current and longest streak; and
  personal records (longest, biggest climb, fastest, best week, …) that open the ride.
- Everything is computed on the phone, in your local time, weeks starting Monday, from a lean query
  (`listRideStats`, ~150 bytes a ride, paged past the API's 1,000-row cap). The logic lives in
  `src/lib/analytics.ts` with 44 unit tests; the charts are in `src/components/charts/`.

**More features**

- **Weekly goal card on Home** — a progress ring toward the target you set in onboarding, with the
  week's days (Monday start) and what's left.
- **Activity** (bell on Home) — kudos, comments, new followers and "rode with" tags, with an unread dot.
- **Bike maintenance** (Garage → "Maintenance" on a bike's card) — add a chain, tyres, brake pads…, give each a
  service interval, and see fresh / soon / due from the bike's odometer. Retiring a part remembers how
  far it lasted.
- **Delete account** (Settings) — a real deletion: the account, rides, routes, comments and uploaded
  photos go (storage files are removed first, since SQL can't), and the phone forgets its local data.
  The Privacy Policy says so, and the website has a public "Delete your account" page for the Play Console.
- A Help row in Settings that opens a pre-filled email to support, and text-length limits on fields
  that match the database.

**One database file.** `mobile/supabase/schema.sql` (v1.5.0) replaces `schema.sql` + `schema_v2.sql` +
`schema_v3.sql`. It is idempotent, builds a fresh project from nothing, upgrades an existing one in
place, and ends with a self-check that rolls the whole run back if a table, view or function is
missing or Row Level Security is off. The last result you see is one row: *Bunch schema 1.5.0 is
installed and up to date.* It was tested on PostgreSQL 16: fresh install, upgrade from the old
three-file layout with data in it, and repeated re-runs.

**The website was rebuilt from scratch** — see `website/README.md`. It is now built around the word
*bunch* and around cycling itself rather than a generic app-landing layout: a big condensed headline
("Ride with the bunch."), a real 32-spoke laced wheel that turns as you scroll, a dictionary entry for
*bunch*, contour-line texture, and the real app screens in phone frames that swap as you move through
the five feature chapters. Pricing is a plain Free-vs-Premium ledger beside the $4.99 card, there is a
public `/delete-account` page (the URL Google Play's data-safety form asks for), a proper phone menu,
and a 1200×630 share image for link previews. The admin console under `/admin` is unchanged.

### Earlier: what 1.4 added

No sample data anywhere (empty states with a next step instead); four labelled tabs; the real Bunch
logo and icon; background recording with a map that follows you; zoomable route maps; Route Detail +
"Ride this route"; Find riders; a fixed elevation-gain calculation; a scrollable Badges screen.

## Setup (fresh Supabase project only)

1. Create a project at [supabase.com](https://supabase.com), open its SQL Editor, paste in the
   entire contents of `mobile/supabase/schema.sql`, and run it once. That's the whole backend.
2. Put that project's URL and anon key into `mobile/.env` and `website/.env` (see each folder's
   `.env.example` for the exact variable names — they differ: `EXPO_PUBLIC_*` vs `VITE_*`).
3. Bootstrap your first admin: sign up for an account from either app (on the website, that's at
   `/admin`), then in the SQL Editor run
   `update public.profiles set is_admin = true where id = '<your-user-id>';` (find your id with
   `select id, username from public.profiles;`). The walkthrough is also at the bottom of schema.sql
   and in `website/README.md`.

## Build the Android APK

```bash
cd mobile
npm install
npm run typecheck     # first real check — see "Verification" below
npm test
```

**Easiest — EAS Build (builds in the cloud, no Android SDK needed):**

```bash
npx eas login                       # free Expo account
# open app.config.ts and replace REPLACE_WITH_YOUR_EAS_PROJECT_ID with a real one —
# `eas build:configure` will create one and do this for you
npm run build:apk:eas
```

EAS gives you a download link for the `.apk` when the build finishes (a few minutes).

> **If the installed app opens on "This build is missing its Supabase keys":** the cloud build
> didn't receive `mobile/.env`. Add the two values as EAS environment variables
> (`eas env:create`, names `EXPO_PUBLIC_SUPABASE_URL` and `EXPO_PUBLIC_SUPABASE_ANON_KEY`,
> visibility *plain text* or *sensitive* — they are public anon values by design) and build again.
> `.env` files are not part of every upload, which is why this screen exists instead of a crash.

**Fully local (needs Android Studio / the Android SDK + a JDK installed):**

```bash
npm run build:apk:local
```

Either way you get an installable `.apk` — side-load it onto a device, or `adb install` it.

This app uses native modules (MapLibre, background location, notifications, Bluetooth), so it
**cannot run in Expo Go** — use a dev client or a built APK. A dev client built before 1.4 needs a
fresh one (`npx expo prebuild --clean`, then rebuild).

## Run the website (marketing site + admin console)

```bash
cd website
npm install
npm run dev     # http://localhost:5173 — marketing site at /, admin console at /admin
```

`npm run build` produces a static `dist/` you can host anywhere (Vercel, Netlify, S3 — it's just
static files hitting Supabase directly, no server of its own). Full feature rundown is in
`website/README.md`.

## How recording works

`src/lib/recording/` is the single owner of an in-progress ride; screens only read from it.

- **`reducer.ts`** — the recording rules, as pure functions: accuracy filter (> 30 m dropped),
  teleport rejection (> 35 m/s between nearby fixes), 1 m jitter floor, distance, max speed,
  auto-pause (below 0.8 m/s for 3 s) and auto-resume (above 1.5 m/s — hysteresis so it doesn't
  flicker), pause/resume/finish, and "interrupted" handling that anchors to the last real fix.
  Covered by `__tests__/reducer.test.ts`.
- **`engine.ts`** — a singleton fed by two sources: the OS background location task
  (`task.ts` + `lib/locationTracking.ts`, backed by an Android foreground service) and an in-app
  fallback watcher used only if background permission is refused. Every batch is written to disk
  (`storage.ts`) before anything else, work is serialised through one promise chain, and state is
  rebuilt from disk after a cold start. A watchdog notices when the OS silently stops delivering
  GPS and marks the ride interrupted rather than inventing moving time.
- **`useRecording.ts`** — the React side (`useSyncExternalStore`), used by the Record screen and
  the tab bar's live indicator.
- **A finished ride stays on disk, untouched, until it is saved or deliberately discarded** — closing
  the app between "Finish" and "Save" cannot lose it. Saving offline queues it (`lib/rideOutbox.ts`)
  and `lib/syncManager.ts` uploads it when there is a connection.
- **`lib/liveLocation.ts`** drives the Record map (1 Hz position + smoothed compass heading, only
  while a screen is on); it is deliberately separate from what gets recorded.
- **`lib/rideNotification.ts`** — the ongoing "Bunch — recording your ride" notification with a live
  timer on Android (posted once per state change on iOS, which has no equivalent for a local
  notification; a true lock-screen timer there needs a Live Activity, i.e. a native WidgetKit
  extension, which is out of scope for this Expo-managed project).

Two things no code can fix, because they aren't bugs: **force-quitting Bunch from the recent-apps
switcher during a ride stops tracking** (the OS cannot resume a background location session after
that — true of every cycling app), and **some Android makers (Tecno, Infinix, Xiaomi, Huawei, parts
of Samsung/OnePlus) kill background services regardless of what the app does**. Setting Bunch's
battery to "Unrestricted" fixes it; Settings → Location has a tip with a shortcut, and the Record
screen walks riders through it the first time.

## Brand assets

| File | Used for |
| --- | --- |
| `mobile/assets/icon_square.png` | iOS / store icon — 1024 px, opaque, full-bleed (the OS rounds it) |
| `mobile/assets/adaptive_icon_foreground.png` | Android adaptive icon — mark sits inside the 66 dp safe circle of the 108 dp canvas, so round / squircle / teardrop masks never clip it |
| `mobile/assets/adaptive_icon_monochrome.png` | Android 13+ themed icon |
| `mobile/assets/splash_mark.png` | Splash (same `#0E0E12` as the icon background) |
| `mobile/assets/notification_icon.png` | Status-bar glyph — white stencil on transparent, as Android requires |
| `mobile/assets/wordmark_dark.png`, `wordmark_light.png` | Lockup for light / dark backgrounds (Home header, Welcome, share card) |
| `website/public/{logo,logo-light,mark,mark-light,favicon}.svg`, `apple-touch-icon.png`, `icon-192/512.png`, `favicon-32/48.png`, `site.webmanifest` | The website |

The in-app theme is light (`src/theme/colors.ts`: accent `#4D47FC`, ink `#15171B`, cream `#FAFAF8`);
the dark splash opening into a light app is intentional.

## Before you charge anyone real money

A few things were deliberately simplified or left as placeholders to get a complete, working app
in front of you — worth a pass before a public launch:

- **The "Support Bunch" donation link is a placeholder.** `mobile/src/lib/constants.ts`'s
  `SUPPORT_ME_URL` points at `https://www.buymeacoffee.com/REPLACE_ME`. Settings hides that row until
  the URL stops containing `REPLACE_ME`, so a build never ships a dead link. Swap in a real Buy Me a
  Coffee / Ko-fi / GitHub Sponsors link when you have one.
- **Legal pages are mostly templates.** `mobile/src/screens/legal/{PrivacyPolicyScreen,TermsScreen}.tsx`
  (and their website twins) have real, specific content (location data, Premium billing, cycling
  assumption-of-risk, account deletion). The support email is real (`adongoabc@proton.me`), but
  `[Legal entity name]`, `[address]` and `[Governing jurisdiction]` are still bracketed placeholders.
  Fill those in and have an actual lawyer read them before you rely on them.
- **Premium isn't sold in-app yet.** Premium is granted by an admin (the Premium screen says so);
  there is no App Store / Play billing flow. Wire RevenueCat or Play Billing before you charge.
- **No native date picker.** Challenge creation on mobile uses duration presets (1 week / 2 weeks
  / 1 month / 3 months starting today) instead of a calendar. The website's admin challenge form does
  use real date inputs.
- **Bluetooth sensor pairing works, but isn't wired into ride recording yet.** The Devices screen
  does real BLE scanning/pairing (`mobile/src/lib/ble.ts`) for heart rate, speed/cadence and power
  and shows a live reading there. Piping `subscribeToSensor()`'s callback into the recording
  session is the natural next step.
- **Google Sign-In needs two dashboard steps before it works.** The app calls
  `supabase.auth.signInWithOAuth({ provider: 'google' })` (`mobile/src/lib/googleAuth.ts`); nothing
  in this codebase touches a Google client secret. (1) In Google Cloud Console create an OAuth
  **Web application** client and add `https://<your-project-ref>.supabase.co/auth/v1/callback` as
  an authorized redirect URI; (2) in Supabase enable **Authentication → Providers → Google** with
  that client's ID/secret, and add `bunch://auth-callback` under **Authentication → URL
  Configuration → Redirect URLs**. Until then "Continue with Google" fails with a clear error.
- **Segments are carved from a ride's full start-to-finish**, not a user-drawn sub-section. A proper
  segment editor (drag start/end points on the map) is the natural next step.
- **Elevation comes from the phone's GPS altitude**, which is noisy on flat roads. The smoothing
  removes most phantom climbing, but a barometer (`expo-sensors`) or a DEM lookup would be more
  accurate still.
- **The website's share image needs an absolute URL once you have a domain.** `website/index.html`
  points `og:image` / `twitter:image` at `/og.png` (relative) because the domain isn't known yet, and
  most link-preview scrapers won't resolve that. Change the two tags to `https://yourdomain.com/og.png`.
  The same domain gives you the Play Console account-deletion URL: `https://yourdomain.com/delete-account`.
- **The yearly price.** The app and site quote $55.09/year (8% off twelve months). If you meant a
  round 10% off ($53.89), change `PREMIUM_YEARLY_PRICE` / `PREMIUM_YEARLY_DISCOUNT` in
  `website/src/lib/constants.ts` and the matching constants in `mobile/src/lib/pricing.ts`.

## Verification — what has and hasn't been run

This was built in a sandbox with no access to the npm registry and no device, so:

- **Not run:** `npm install`, `expo prebuild`, a native build, or the app on a phone or emulator.
  Run `npm install && npm run typecheck && npm test` in `mobile/` (and `npm install && npm run
  build` in `website/`) as the first real check. Anything that surfaces should be small and
  mechanical — a wrong prop name, a missing type — not architectural.
- **Run:** the app's `src` was type-checked without the real library typings (so that catches
  missing names, wrong imports and bad exports, not prop-level mistakes) and every `@/` import was
  resolved. The pure logic has unit tests — `lib/analytics.ts` (44), `lib/geo.ts`, `lib/units.ts`,
  `lib/goal.ts`, `lib/authRedirect.ts` and the recording rules in `lib/recording/reducer.ts`
  (`npm test`, Jest via `jest-expo`) — plus a headless run of the recording engine (start, ride,
  compaction, process restart, OS kills the task, finish, refuse a second start) that is not shipped
  in this folder. `schema.sql` was run against a real PostgreSQL 16 (fresh, upgrade, re-run, RLS).
- **Rendered:** screens were rendered in a browser-based preview (the real `src` code with React
  Native swapped for DOM shims and a stand-in map) at phone width and inspected. That checks layout,
  spacing, empty states, charts and navigation flow — not native map behaviour, gestures, GPS or
  background execution, which need a device. The website was rendered the same way in Chromium at
  phone, tablet and desktop widths.

## Design

Both apps share one visual system (`mobile/src/theme/colors.ts` is the source of truth; the
website's `tailwind.config.ts` mirrors it): a light, warm-neutral canvas, one indigo accent, Manrope
for text and tabular figures that look like they came off a bike computer. The public website adds
one display face on top of that — Big Shoulders, a condensed bold, self-hosted with its licence in
`website/public/fonts/` — for headlines and big numbers. The layout language
follows the reference screens — labelled bottom tabs, grouped settings lists, label-above-value
stats, a bottom sheet on the Record map. Charts use one data colour (the accent) with a pale tint
for context, thin marks, hairline grid and a table view for every chart. Onboarding and auth use real
entrance animations (`react-native-reanimated`); sign-in and sign-up are one screen with a segmented
tab that slides between the two forms.

# bunch-cycling
