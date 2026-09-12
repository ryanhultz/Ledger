# STOP — v1.11.0 Update Guide (Security & Reliability Release)

This is a bigger update than usual — the login/permission system now
does real server-side verification instead of trusting the app's UI,
plus a service worker for faster loading and error reporting. It
needs a bit more one-time setup than past updates, but nothing here
changes how you use the app day-to-day.

## What's in this zip

- `index.html` — the app itself
- `worker.js` — the updated Worker script (now handles login, signup,
  and permission checks — not just data sync)
- `service-worker.js` — **new file**, deploy this alongside
  `index.html` in the same folder (not inside the Worker)
- `README.md` — this file

## What changed, and why it needed new setup

**Before:** the app fetched the entire user list (including
passwords) to the browser and checked logins there. The master
password and invite code were sitting in plain text in `index.html`
— anyone could view-source the page and read them.

**Now:** the Worker itself checks passwords, issues a signed session
token, and enforces who's allowed to write what — before it ever
touches the shared data. Passwords never leave the server. The master
password and invite code live only on the Worker, as encrypted
Secrets, never in a file you deploy.

## Setup steps

### 1. Update the Worker code
1. Go to [dash.cloudflare.com](https://dash.cloudflare.com) → **Workers & Pages** → open your existing `stop-sync` Worker.
2. Click **Edit code** (or **Quick edit**).
3. Select all the existing code, delete it, and paste in the entire contents of the new `worker.js` from this zip.
4. Click **Save and deploy**.

Your KV binding (`STOP_KV`) and Cron Trigger both carry over automatically — nothing to redo there.

### 2. Add three Secrets to the Worker (new — required)
The master password and invite code now live here instead of in the app file.

1. On your Worker's page, go to **Settings → Variables**.
2. Under **Secrets** (sometimes labeled "Environment Variables" with a "Secret" toggle), click **Add**.
3. Add three secrets, one at a time:
   - **Name:** `ADMIN_PASSWORD` → **Value:** whatever you want the master password to be (e.g. your current `1111`, or something stronger — this is a good time to pick a better one)
   - **Name:** `INVITE_CODE` → **Value:** whatever you want the invitation code to be (e.g. your current `newusers1234`, or a new one)
   - **Name:** `AUTH_SECRET` → **Value:** any long random string, e.g. `k9J3xQ7vM2pL8wR4tY6nB1cF5hD0sA9e` (mash the keyboard or use a password generator — this one just needs to be long and secret, nobody ever types it in; it's what the Worker uses to sign login sessions so they can't be faked)
4. Save each one. The Worker redeploys automatically.

**Important:** if you skip this step, logins and the master password screen will simply fail with "Incorrect password" / "Invalid invitation code" for everyone, since the Worker has nothing to check against.

### 3. Deploy the new index.html
Deploy it exactly the way you already do (Cloudflare Pages / GitHub). No change needed to `SYNC_API_BASE` — same Worker URL as before.

### 4. Deploy service-worker.js alongside index.html (new file)
This needs to sit in the **same folder** as `index.html` at your site's root — not inside the Worker, not in a subfolder. If you're using:
- **Cloudflare Pages Direct Upload:** drag both `index.html` and `service-worker.js` into the same upload.
- **GitHub → Cloudflare Pages:** commit `service-worker.js` to the repo root, right next to `index.html`.

If it's missing, the app still works completely normally — it just won't get the faster-load caching or Android's install prompt.

### 5. Have everyone log out and back in once
Because sessions now come from the server, everyone's next login will issue them a proper token. Nothing needs to be done manually — logging in normally handles it. Existing accounts, permissions, and passwords are untouched.

## What else changed in v1.11.0 (no setup required)

- **"Today's Schedule" moved to the top of the Home page.**
- **Allergy Lookup now groups ingredient variants automatically** — e.g. "Lavandula Angustifolia (Lavender)," "...Flower Extract," and "...Oil" all show up as one searchable "Lavender" entry instead of three separate ones. No manual grouping needed; it matches by botanical name first, common name as a fallback.
- **Error reporting** — if the app hits an unexpected error, it now shows a plain "Something went wrong — Reload App" screen instead of quietly breaking, and logs what happened to a new **Error Log** section in Management (permission-gated like everything else) so you can see what broke, for whom, and when — without needing to be standing there when it happens.

## Notes

- Passwords are still stored as plain text in the database (not
  hashed) — the improvement here is that the *server* now verifies
  them and enforces permissions, instead of just trusting whatever
  the browser claims. Real password hashing is a reasonable next
  step, but wasn't part of this update. Don't reuse a real personal
  password for any account here.
- Session tokens last 24 hours for regular users, 4 hours for the
  master password — separate from, and in addition to, the app's
  existing midnight auto-logout.
- If the Worker is ever unreachable, the app falls back to the last
  data it saw on that device and shows a warning banner — logins,
  though, do require a live connection, same as before.
- The Cron Trigger and KV namespace from earlier setup are untouched
  by this update.

---

## v1.12.0 update — user storage rebuilt (fixes the login/reset bug)

If you've been hitting a bug where resetting one person's password (or
logging in) seemed to randomly break a *different* account, this
update fixes it structurally, not just reduces the odds of it.

**What changed:** every user used to live together in one shared list
in the database. Any single action — a password reset, a new
signup, even just logging in — had to read that whole list, change
one part, and save the whole thing back. If two of those happened
close together, one could silently overwrite the other's change,
since Cloudflare's database doesn't guarantee a "moment ago" read is
still current. Now, **each person has their own separate, independent
record.** Resetting Test's password only ever touches Test's own
record — there's no shared list left for it to accidentally disturb.

**Setup for this update:**
1. Paste the new `worker.js` into your Worker (same as always — Edit
   code → select all → replace → Save and deploy). Your Secrets, KV
   binding, and Cron Trigger all carry over untouched.
2. Deploy the new `index.html` the same way you always do.
3. **No manual data migration needed** — the very first time the
   Worker needs the user list after this update, it automatically
   splits your existing users into the new format, once, in the
   background. Nothing to click, nothing to configure.
4. Test in a private/incognito window the first time, so you know
   you're seeing the new code and not a cached copy of the old page.

Everyone's usernames, passwords, and permissions carry over exactly
as they are today — nothing needs to be re-entered.

---

## v1.14.0 update — Rooms, Durations, Categories, and Home Links are now yours to edit

Four things that used to only be changeable by asking for a code
update are now fully in your hands, in a new **Management → Settings**
section.

**Setup for this update:**
1. Paste the new `worker.js` into your Worker (Edit code → select all
   → replace → Save and deploy). Secrets, KV binding, and Cron
   Trigger all carry over untouched.
2. Deploy the new `index.html` the same way you always do.
3. **No manual setup beyond that.** The first time the app loads
   after this update, it automatically fills in your current Rooms,
   Durations (50/100), and Home Links (Agilysys, Breakroom) into the
   new editable lists — nothing to re-enter, nothing to configure.

**What's new:**
- **Rooms** — add, rename, or remove rooms yourself.
- **Durations** — add appointment lengths beyond 50/100 minutes. The
  shortest configured duration is the default for new slots; any
  longer one still takes up the next time slot, the same way 100-min
  appointments always have.
- **Categories** — add new Protocol categories (like "Facials")
  beyond the built-in Body Treatments / Kurr Collection / Massages /
  Custom — they show up in the Protocol form right away.
- **Home Links** — the Agilysys and Breakroom buttons on Home are now
  regular, editable entries instead of fixed code — rename them,
  change where they point, or add entirely new ones (website or app).

This is all gated behind a new **Settings** permission, same pattern
as everything else in Management — grant it to whoever should be able
to make these changes.

---

## v1.15.0 update — Sub-Protocols and Shift Templates

**Setup for this update:**
1. Paste the new `worker.js` into your Worker (Edit code → select all
   → replace → Save and deploy). Secrets, KV binding, and Cron
   Trigger all carry over untouched.
2. Deploy the new `index.html` the same way you always do.
3. No manual setup beyond that — both new lists just start empty and
   are ready to use.

**What's new:**

- **Sub-Protocols** (Management → Protocols → Sub-Protocols) — a
  reusable, named step-sequence, the same idea as the built-in Facial
  Massage routine, but ones you create yourself. Build one once, then
  use "Insert Sub-Protocol" in any main Protocol's Steps to drop it
  in. The key part: it's a **live reference**, not a copy — edit or
  rename the Sub-Protocol later, and every Protocol that uses it
  updates automatically, with nothing to touch on the Protocols
  themselves. If a Sub-Protocol is ever removed, any Protocol still
  referencing it shows a plain "not found" note instead of breaking.

- **Shift Templates** (Management → Settings → Shift Templates) —
  save common start/end time pairs (like "Morning" or "Closer"). On
  the Agenda setup screen, a "Quick Pick a Shift" dropdown appears
  whenever you have at least one saved, filling in both times in one
  tap instead of picking them from scratch each time.

---

## v1.16.0 update — Renamed to The Ledger, plus a large feature batch

**Setup:**
1. Paste the new `worker.js` into your Worker (Edit code → select all → replace → Save and deploy). Secrets, KV binding, and Cron Trigger all carry over untouched.
2. Deploy the new `index.html` the same way you always do.
3. No manual data setup — the new protocol/products import automatically, existing accounts are untouched, and the new self-service fields are simply blank until someone fills them in.

**What's new:**

- **Renamed to "The Ledger"** everywhere it's user-facing — purely cosmetic, nothing to configure. (The internal `STOP_KV` binding name and other code-only identifiers are unchanged on purpose.)

- **Fixed a real bug**: creating a new account would sometimes get logged straight back out immediately after signup. This was caused by the app double-checking a brand-new account against a "list everyone" query that can briefly lag behind Cloudflare's database under the hood. Login and signup now trust the server's own confirmation immediately instead.

- **Self-service Change Password** — a new **My Account** button next to Log Out lets anyone change their own password without needing an admin.

- **Security Question password recovery** — set a recovery question when creating an account (or add one later from My Account). A "Forgot password?" link on the login screen lets you answer it and set a new password yourself, no admin needed. Accounts with no question set are told plainly to contact an admin instead.

- **Users list** in Management is now a plain list instead of pill-shaped buttons.

- **Sub-Protocol insertion** now drops the reference wherever your cursor is in the Steps box, instead of always jumping to the end.

- **Prep list "Choice Groups"** — a Prep row can now hold multiple product alternatives (e.g. "guest's choice of Restore, Relief, Revive, or Relax Oil") sharing one amount. The Allergy Lookup automatically treats a Choice Group as unsafe if **any** alternative contains the allergen — no extra setup needed for this, it works through the same matching the app already does.

- **SOAP Notes** — a new Home button for writing a session note (Subjective/Objective/Assessment/Plan) and turning it into a PDF. The draft lives only on that device's browser storage — it's never sent to or stored in the shared database. Generating the PDF opens a clean printable page and uses your browser's own Print → Save as PDF — this avoided bundling a whole PDF-generation library into the app just for this one feature. The draft clears itself once the PDF is generated.

- **New protocol + 4 products** — Restorative Therapeutic Massage, plus the Restore/Relief/Revive/Relax Massage Oils (Mirbeau's "Monet's Favorite Fragrance" line), imported automatically.

## Notes

- Security question answers are stored in plain text, same tier of care as passwords — not shown to anyone browsing the Users list, but not hashed either.
- SOAP Note drafts persist across logging out and back in on the same device (since nobody shares devices here) — they are not tied to a specific login session.

---

## v1.17.0 update — Demo Account

**Setup:**
1. Paste the new `worker.js` into your Worker. Secrets, KV binding, and Cron Trigger all carry over untouched.
2. Deploy the new `index.html`.
3. The demo account creates itself automatically — no manual setup. The first time someone with Users access loads the app after this update, a `demo` / `demo2026` account is created with every permission enabled and Demo Mode on.

**What it does:**

Log in as `demo` / `demo2026` and everything works completely normally — real data, real screens, every save appears to succeed — but nothing that account touches ever actually reaches the shared database. Add a product, reset someone's password, restore a backup, change your own password — all of it behaves like it worked, and none of it persists. Log out, and it's as if nothing happened. Nobody else using the app at the same time is ever affected by it either, even if they're making real changes at that exact moment.

This is controlled by a single "Demo Mode" toggle on any user, in Management → Users → (select a user). Any account can be flagged this way, not just the built-in `demo` one.

**Scope note:** this is intended for internal use — showing a new hire around, or testing something yourself without worrying about leaving a mess. It shows the *real* data (real products, real staff usernames, real Agenda entries) — it does not create fake/sample content to look at. If this ever needs to be shown to someone outside your own team, that's a different, bigger project (either masking real data or a fully separate demo environment) — worth a separate conversation if that need comes up.

---

## v1.19.0 update — Security hardening + renameable categories

**Setup:**
1. Paste the new `worker.js` into your Worker. Secrets, KV binding, and Cron Trigger all carry over untouched.
2. Deploy the new `index.html`.
3. No manual data migration needed — existing accounts, categories, and everything else upgrade themselves automatically, in place, the first time they're used.

**What's new:**

- **Passwords are now hashed, not stored in plain text.** This was the single biggest gap in the app's security — closed. Every existing account upgrades itself automatically and invisibly the next time that person logs in successfully with their current password; nothing needs to be reset, nobody will notice anything changed.

- **Demo accounts are now enforced by the server, not just the app.** Previously, "nothing a demo account does ever saves" was purely a front-end behavior — someone bypassing the app entirely and calling the API directly with real demo credentials could have actually written real data. The Worker itself now refuses every write from a demo-flagged account, including the destructive ones like Backup Restore — proven with a direct test that simulates exactly that bypass attempt.

- **New self-signup accounts start in Demo Mode automatically.** Anyone creating an account through the invite-code signup screen gets a fully-usable account that can't actually change anything, until an admin explicitly turns Demo Mode off for them in Management → Users. Accounts an admin creates directly (Add User) are unaffected and start as real accounts, same as always.

- **Built-in Protocol categories can now be renamed.** Body Treatments, Kurr Collection, Massages, and Custom can each be given a new display name in Management → Settings → Categories. Protocols already filed under one stay correctly grouped — only the label changes, not the underlying data.

## Notes

- Password hashing uses PBKDF2 (100,000 iterations, SHA-256, random per-user salt) via the Web Crypto API already built into Cloudflare Workers — no new dependency.
- If you're curious whether the demo-enforcement fix actually works: a demo account with every permission granted was used to directly attempt overwriting real product data, deleting a real user, and restoring a backup — all three were silently absorbed with no actual effect on real data.

---

## v1.20.0 update — CORS locked to your actual domain

**Setup:**
1. Paste the new `worker.js` into your Worker. It works immediately with no
   further setup — it already knows your current domain
   (`https://spa.ryanhultz.workers.dev`) as a safe default.
2. Deploy the new `index.html` (unrelated changes bundled into this release —
   see below).

**What changed:** the Worker used to respond to requests from *any*
website (`Access-Control-Allow-Origin: *`). It now only responds to
your actual app's domain — a request from a random other website gets
turned away.

**If you ever change the app's domain:** you do **not** need a new
code deploy for this. Go to the Worker's **Settings → Variables**,
add or edit `ALLOWED_ORIGINS` with the new domain (comma-separate
multiple domains if you ever need more than one, e.g. a staging URL),
and save. Forgetting this step after a real domain change is the one
way this could quietly break the app — worth remembering if a domain
change is ever on the horizon.
