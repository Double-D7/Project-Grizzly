# Project Grizzly — Cloud Sync & Apple Health: Design & Evaluation

_Design deliverable for Stage 6. Nothing here is wired up yet — this is the plan you
approved me to draft now and implement later. No external accounts or services have
been created._

---

## Part 1 — Supabase Cloud Sync

### Goal
Let David open Grizzly on a new phone, a reinstall, or a second device and have every
weigh-in, workout, nutrition day, check-in, photo, readiness log and setting already
there — without ever losing the offline-first behaviour the app has today.

### Guiding principles
- **Offline stays the source of truth.** The app keeps working with no network, exactly
  as now (localStorage + the IndexedDB vault + daily snapshots). Cloud is a *mirror and a
  recovery channel*, not a dependency.
- **One account, your data only.** Row-Level Security so a row is readable/writable only
  by the user who owns it.
- **No silent data loss.** Sync is additive and checksum-guarded, the same discipline as
  the file backup/restore already shipped.

### Recommended architecture: "state document" sync (not normalized tables)
The whole app state is already one JSON object (`S`) with a `schemaVersion` and a working
FNV-1a checksum. The cheapest, safest sync is to store that object as a single row per user
rather than shredding it into 8 relational tables.

**Table `grizzly_state`**
| column | type | notes |
|---|---|---|
| `user_id` | uuid (PK) | = `auth.uid()` |
| `state` | jsonb | the full `S` object (minus photos) |
| `schema_version` | int | for server-side guard |
| `checksum` | text | FNV-1a of `state`, same fn as the file export |
| `updated_at` | timestamptz | last write wins tiebreaker |
| `device` | text | last device that wrote (for the UI) |

**Photos** are too big for a JSON row, so they go to **Supabase Storage** (a private bucket
`progress-photos/{user_id}/{photoId}.jpg`). The state row keeps only photo *metadata*
(id, pose, date), which it already has in IndexedDB. Photos upload/download lazily.

**Why one document, not tables:** the data is single-user and read/written as a whole on
every `save()`. A JSONB row means zero migration work when the schema changes (the app's
own `migrate()` already handles version bumps on load), one RLS policy, and trivial
conflict logic. Normalized tables would buy query power David doesn't need and cost real
migration risk every stage.

### Auth
- **Supabase Auth, email magic-link** (passwordless). One tap from the email, no password
  to manage. Free tier covers this comfortably for a single user / small circle.
- Until signed in, the app behaves exactly as today (local only). Sign-in is optional and
  framed as "Turn on cloud backup", not a gate.

### Sync strategy
- **Pull on load (when signed in + online):** fetch the row, compare `updated_at` and
  `checksum` to local. If remote is newer, run it through the local `migrate()` and merge.
- **Push on change:** wrap the existing `save()` to also enqueue a **debounced** push
  (e.g. 3–5 s after the last edit) so we're not hammering the network per keystroke.
- **Offline queue:** if a push fails (offline), set a dirty flag; flush on the next
  `online` event. This reuses the pattern already in place for the vault mirror.
- **Conflict policy — last-write-wins with a safety net:** `updated_at` decides the winner.
  Because two devices editing the *same day's* fields within seconds is the only real
  collision for a single user, LWW is sufficient. The safety net: before any overwrite we
  already keep daily local snapshots + file backups, so a bad merge is always recoverable.
  (A future upgrade could do per-collection field merging; not needed for v1.)
- **Integrity:** every push/pull validates the FNV-1a checksum, same as file import. A
  mismatch aborts the merge and surfaces a toast rather than corrupting local data.

### Code seam (how it drops into the current app)
Minimal, non-invasive:
1. A `cloud.js`-style module with `signIn()`, `pull()`, `pushDebounced()`, `uploadPhoto()`.
2. One line inside the existing `save()` → `cloud.pushDebounced()` (no-op when signed out).
3. One call in the boot sequence → `cloud.pull()` after the local load + vault-recover step.
4. A "Cloud backup" card in the existing Data Safety section: sign-in state, last sync
   time, and a manual "Sync now".

Everything else — the UI, the analytics, the offline flow — is untouched.

### Cost / privacy
- Supabase **free tier**: 500 MB database, 1 GB storage, 50k monthly active users. A single
  operator's text state is kilobytes; photos at ~150 KB each means thousands fit in 1 GB.
  **No paid API required** for this scope.
- Data is private per RLS. Photos in a private bucket with signed URLs. Nothing is public.
- This needs: a Supabase project + its URL and anon key dropped into the app config. I have
  **not** created any of that — it's the first implementation step whenever you say go.

### Rollout plan (when approved to build)
1. Create the Supabase project; add `grizzly_state` + the photos bucket + RLS policies.
2. Ship the `cloud` module behind a hidden flag; test sign-in + pull/push on two browsers.
3. Add the Data Safety "Cloud backup" card; verify offline → online flush.
4. Verify a fresh install restores fully (state + photos) from cloud.

---

## Part 2 — Apple Health: evaluation

### The hard constraint
A website / PWA **cannot read Apple Health directly.** HealthKit has no web API, and Safari
exposes none of it to web pages. So "Grizzly auto-pulls my steps/weight/sleep from Health"
is **not possible while Grizzly is a PWA.** Anyone who claims a pure web app does this is
either using a native wrapper or a manual import. Options, honestly ranked:

| Option | What it does | Effort | Needs native app? | Verdict |
|---|---|---|---|---|
| **A. Apple Shortcuts → HTTP** | A Shortcut reads Health (steps, weight, sleep) and POSTs to a Supabase endpoint on a schedule/automation | Low–medium | No | **Recommended** once cloud sync exists |
| **B. Manual Health Export import** | User exports `export.zip` from the Health app; Grizzly parses the XML for weight/steps/workouts | Medium (big XML parse) | No | Good one-time / occasional catch-up |
| **C. Native wrapper (Capacitor/Swift)** | Real HealthKit read/write via a thin iOS app wrapping the web UI | High | **Yes** | Only if you want a true App Store app later |
| **D. Third-party bridge** | Services that relay Health data | — | varies | Usually paid / privacy cost — not recommended |

### Recommendation
- **Short term:** do nothing native. Grizzly already captures the metrics that matter
  (weight, steps, sleep, RHR) with fast manual entry, and the analytics are built on those.
- **Best fit once cloud sync is live (Option A):** an **Apple Shortcut** David runs (or
  automates each morning) that reads *today's weight and steps* from Health and POSTs them to
  a tiny Supabase function. Grizzly then shows them on next sync. This is free, needs no
  native app, and no App Store review. I can write the Shortcut recipe + the endpoint when
  cloud sync is in.
- **Option B** is a nice "import my history" one-off if David has months of Health data he
  wants pulled in at once — a dedicated importer for the Health `export.zip`.
- **Option C** is the only path to true two-way HealthKit and is a separate project (turn
  the PWA into a Capacitor app). Worth it only if an App Store presence becomes a goal.

### What I'd build first (when approved)
1. Finish Supabase cloud sync (Part 1).
2. Add a Supabase Edge Function `ingest-health` that accepts `{date, weight, steps, sleep}`.
3. Write the Apple Shortcut recipe that reads Health and calls it.
4. (Optional) a Health `export.zip` importer in the Data Safety section for bulk history.

---

## Summary
- **Cloud sync:** single JSONB state row per user + private photo bucket, magic-link auth,
  last-write-wins with checksum guard and an offline queue, built as a small `cloud` module
  that hooks `save()` and boot. Free tier. Offline stays the source of truth.
- **Apple Health:** not possible from a pure PWA directly; the clean, free path is an Apple
  Shortcut POSTing to Supabase (after sync exists), with an optional Health-export importer
  for history. True HealthKit needs a native wrapper and is a separate, later project.
- **Nothing external created yet** — all of this is ready to implement on your word.
