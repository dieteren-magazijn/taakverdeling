# Handover

What someone taking over this application needs to know. Written for a
maintainer who has never seen it before.

For what changed in each version, see `CHANGELOG.md`. For how to build and
deploy, see `README.md`.

## 1. How critical is it

Medium. The board organises the day; it does not control any process.

If it stops working, the team leaders fall back to what they did before it
existed: a printed roster and a whiteboard. Nothing in the warehouse halts. In
practice a failure is noticed within minutes, because between five and ten
people are looking at it.

The most likely failure is not a bug but the database being unreachable. The
application handles that by keeping the last known state on screen and marking
the connection as lost. It does not silently show stale data as if it were
current.

## 2. Architecture in one page

There is no server of our own. Three parts:

1. **The page.** Static HTML and JavaScript on GitHub Pages. No build step. The
   browser does all the work.
2. **The database.** A hosted Postgres (Supabase). It stores the entire board
   as a single JSON document and nothing else. There is no application logic in
   the database.
3. **The roster Excel.** Read locally on the planning PC through the File
   System Access API. It never leaves that machine and is never uploaded.

Devices have a role, stored locally per device (`vp_rol`, never synced):

- A **PLAN PC** may link the Excel and publish the planning.
- An ordinary device is a pure database client.
- A **screen** runs in board mode with the read only account.

This split exists so that a TV screen or a random PC can never overwrite the
planning.

## 3. The database

```
public.planbord ( id text primary key, doc jsonb, updated timestamptz )
```

Two rows are in use: `hoofdbord` (the planning) and `coli_focus`.

**Access model.** Row level security is enabled. Policies allow select, insert
and update for the `authenticated` role only. The kiosk account is held to read
only by an email check inside the insert and update policies, so the screens
can log in but cannot write.

Anonymous access is not granted. The publishable key that appears in the source
is public by design and grants nothing on its own.

**Accounts.** Individual logins per person in the form
`firstname.lastname@dieteren.be`, plus one shared read only kiosk account for
the screens. Passwords live in a password vault, never in the repository.

**The screens log in through the URL.** The Appspace displays open a link with
base64 encoded kiosk credentials in the URL fragment. This is weak by design
and acceptable only because the account is read only. If that account is ever
given write access, this becomes a real problem.

**Data budget matters here.** The free plan has a monthly data allowance and an
earlier instance of this database was suspended for exceeding it. The polling
optimisation described in `README.md` is what keeps it under. Roughly 7 MB per
hour is the working budget. Before adding any new polling, calculate its cost.

## 4. What personal data it holds

Whoever owns this needs to know, because it decides where it is allowed to run.

- Names of warehouse staff.
- Working hours per person, including per weekday exceptions.
- Absence, including the reason: holiday, illness, time credit, training.
- Fire warden and first aid qualification per person.

Absence reasons are shown on the editing screens but not in board mode; the TV
screens show only that someone is absent. That distinction is deliberate and
should be preserved.

An open item is proper role based access to absence detail. Today anyone who
can edit can see the reason.

## 5. Known pitfalls

These have each cost a day at least once.

**Adding a synced key requires four changes, not three.** The dirty flag in
`onVpStoreSet`, then `collectLocal()`, then `syncRead()`, and finally the `doc`
object in `syncWrite()`. That last one is a fixed list of picked fields.
Anything not in that list is never written and stays local, which looks exactly
like a sync bug.

**The version number lives in three places.** Miss one and devices will not
pick up the new version, or will reload in a loop.

**Line endings are mixed** (CRLF in the main script region, LF in later
modules). Multi line search and replace across that boundary silently fails.
Use unique single line anchors.

**Validate before pushing.** The syntax check in `README.md` must report zero
errors. There is no build step and no test suite, so that check is the only
safety net between an edit and the live tool.

**A failed read must never trigger a write.** Earlier versions would write out
a half loaded state after a failed read, wiping the board for everyone. The
guard is in `syncWrite`; do not remove it.

**Board mode hides the editing controls with CSS.** If a control is invisible,
check whether it is hidden by board mode or by the admin toggle before assuming
it is broken.

## 6. Ownership and open items

These are the things that should not stay as they are. They are listed here
rather than hidden because they are the main reason this document exists.

**Hosting.** The pages are served from GitHub Pages under a GitHub organisation
created by the original author, not under a company account. The repository is
public. No credentials are in it and that has been verified across the full
history, but the hosting itself sits outside company control and should move.

**The database.** It runs on a free tier plan under a personal account. It
holds the personal data listed in section 4. This is the item with the
strongest case for moving to company ownership, and the failure mode is not
theoretical: the previous instance was suspended once for exceeding the free
allowance.

**Single maintainer.** One person knows this codebase. The remedy is not
documentation alone; it is a second person walking through it while the first
is still available.

**Excel dependency.** The day roster comes from an Excel file that is prepared
elsewhere. If its layout changes, the import breaks. There is no contract or
schema on that file and no one has been asked to keep it stable.

**Open feature work.** Publishing the imported roster to the database so that
devices without the Excel link see everything, and role based access to absence
detail.
