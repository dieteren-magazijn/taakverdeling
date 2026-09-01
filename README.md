# Taakverdeling (warehouse day planning board)

Web application for the daily task planning of the D'Ieteren parts warehouse in
Kortenberg. Team leaders assign people to departments and tasks; the result is
shown live on TV screens on the warehouse floor.

In daily use. Two audiences: a handful of people who edit the planning, and a
set of read only screens that display it.

## What it does in practice

- A team leader opens the board on a PC, loads the day roster from an Excel
  file, and drags people onto departments and tasks.
- Every change is written to a shared database, so a second PC sees it within
  seconds.
- TV screens run the same page in board mode with a read only account. They
  show the planning only, never the editing controls.

## Repository layout

| Path | What it is |
|---|---|
| `beta/index.html` | The active application. All work happens here. |
| `beta/colisage.html` | A second board for the sorting aisle. Fed separately. |
| `beta/tizen.html` | Diagnostic page for TV screens, written in ES5 on purpose. |
| `index.html` | The previous production version (v3.69), file based sync over SharePoint. Barely used, kept for reference. |

## Technical shape

Deliberately simple, because there was no build server and no deployment
pipeline available:

- One large HTML file with inline script modules. No framework, no build step,
  no package manager. Plain JavaScript.
- One external library: SheetJS 0.18.5 from cdnjs, used to read the roster
  Excel.
- The roster Excel is read locally through the File System Access API plus
  IndexedDB, which is why the editing side needs a Chromium based browser. The
  read only board side works anywhere.
- Shared state lives in a hosted Postgres database. See "Data and sync".

Line endings are mixed: the main script region is CRLF, later modules are LF.
Be careful with multi line edits and prefer unique single line anchors.

## Hosting

Served as static files from GitHub Pages:

- Production: https://dieteren-magazijn.github.io/taakverdeling/
- Active version: https://dieteren-magazijn.github.io/taakverdeling/beta/

This hosting is a stopgap and is expected to move. See `HANDOVER.md`.

## Data and sync

A single table holds the whole board as one JSON document per row:

```
public.planbord ( id text primary key, doc jsonb, updated timestamptz )
```

Rows in use: `hoofdbord` (the planning itself) and `coli_focus`.

Sync is deliberately frugal. The application polls only the `updated` timestamp
every 7 seconds and downloads the full document only when it has actually
changed. That keeps the data volume near 7 MB per hour. An earlier version did
not do this: it fetched the entire document on every poll and exhausted its
data allowance, which took the board offline. Do not undo this optimisation.

Access is protected by row level security plus per user logins. The screens use
a separate read only account. Details in `HANDOVER.md`.

## Making a change

1. Edit `beta/index.html` directly.
2. Validate the inline scripts. This must report `fouten:0`:

   ```
   node -e "const fs=require('fs');const h=fs.readFileSync('beta/index.html','utf8');let n=0,e=0;h.replace(/<script(?![^>]*\bsrc=)[^>]*>([\s\S]*?)<\/script>/gi,(m,c)=>{n++;try{new Function(c)}catch(x){e++;console.log('FOUT script #'+n+': '+x.message)}return m});console.log('scripts:'+n+' fouten:'+e)"
   ```

3. Bump the version in three places: `var APP_VERSION="vX.YZb"`, the `LOCAL`
   fallback `||"vX.YZb"`, and `beta/version.txt`.
4. Commit and push. GitHub Pages deploys automatically, roughly one minute.
5. Verify with `.../beta/version.txt?cb=<random>`.

This is a live tool used every day. Always validate before pushing.

## Security notes

- The GitHub repository is public. Never commit passwords, database passwords
  or the `service_role` key. This has been verified across the full history.
- The anon / publishable key is in the source on purpose. Those keys are public
  by design; the protection is the login plus row level security.
- Credentials live in a password vault, never in the repository.

## Further reading

- `HANDOVER.md` for architecture, operations, known pitfalls and open items.
- `CHANGELOG.md` for the version by version history.
