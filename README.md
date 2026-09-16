# Online Scheduler Audit

A public, read-only dashboard that tracks weekly audits of the public
website scheduler and Google Business Profile booking link for 41 dental
office locations across 19 practice groups.

It's a static site — no server, no database, no login. All shared state
lives in one JSON file, [`data/audits.json`](data/audits.json), which this
repository serves straight from GitHub Pages. **No patient information is
collected, displayed, or stored anywhere in this project.**

## What's in this repository

```
index.html                        The dashboard (entry point)
assets/
  styles.css                      All styling (light + dark mode)
  app.js                          Loads data/audits.json, computes status/
                                   diffs, renders everything, handles filters
data/
  audits.json                     THE SHARED DATA SOURCE — registry + audit history
  registry-lock.json              Frozen copy of the protected registry (see below)
schema/
  audits-file.schema.json         JSON Schema for the whole data file
  audit-entry.schema.json         JSON Schema for one audit history entry
scripts/
  validate-data.js                Validates data/audits.json (schema, protected
                                   registry, append-only history)
.github/workflows/
  validate-data.yml               Runs the validator on every push/PR
docs/
  JSON_SCHEMA.md                  Full field-by-field data documentation
  WEEKLY_AUDIT_GUIDE.md           Step-by-step instructions for the weekly audit
```

## How the data model works

- **`practices` and `locations`** in `data/audits.json` are the *protected
  registry* — the 19 practice groups, 41 locations, their scheduler/Google URLs,
  and each location's expected appointment types. This almost never
  changes, and `data/registry-lock.json` is a frozen copy of it that CI
  checks against on every change, so a routine audit update can't
  accidentally rename, remove, or overwrite a location.
- **`auditHistory`** is an append-only log of raw observations, one entry
  per completed audit of one location. A location with no entries yet
  shows as **Not Yet Audited** until its first completed run.
- The dashboard itself computes everything derived — overall status
  (Passed / Warning / Failed / Manual Review), per-appointment-type availability failures and week-over-week change
  summaries — from the raw entries, every time the page loads. Nothing
  derived is stored, so the comparison and status rules are always applied
  consistently. The full rules are documented in
  [`docs/JSON_SCHEMA.md`](docs/JSON_SCHEMA.md).

Updating what every viewer sees is just committing a new entry to
`auditHistory` in `data/audits.json` — see
[`docs/WEEKLY_AUDIT_GUIDE.md`](docs/WEEKLY_AUDIT_GUIDE.md).

## Deploying to GitHub Pages

1. **Create a new GitHub repository** (public, so Pages can serve it, and
   so anyone with the link can view the dashboard) and push this project
   to it:

   ```bash
   cd online-scheduler-audit
   git init                                   # if not already a git repo
   git add .
   git commit -m "Initial commit: Online Scheduler Audit dashboard"
   git branch -M main
   git remote add origin https://github.com/<your-org-or-username>/<repo-name>.git
   git push -u origin main
   ```

2. **Turn on GitHub Pages:**
   - In the repository, go to **Settings → Pages**.
   - Under **Build and deployment**, set **Source** to **Deploy from a
     branch**.
   - Set **Branch** to `main` and the folder to `/ (root)`.
   - Save. GitHub will give you a URL like
     `https://<your-org-or-username>.github.io/<repo-name>/` — that's the
     public link anyone can open to view the dashboard.
   - The first deploy can take a minute or two. Re-deploys after that
     happen automatically on every push to `main`.

3. **(Recommended) Protect the `main` branch** so only reviewed changes —
   and passing CI — land in the live data file:
   - **Settings → Branches → Add branch protection rule** for `main`.
   - Require a pull request before merging.
   - Require the `validate` status check (from
     `.github/workflows/validate-data.yml`) to pass before merging.
   - This means: anyone with write access to the repo can propose a data
     update via a PR, but it can't merge (and can't go live) unless the
     protected registry is intact and audit history was only appended to.
   - Only people/integrations with write access to the repository (e.g.
     your connected GitHub account, or a GitHub App/PAT you grant access
     to) can ever change `data/audits.json` — anonymous dashboard viewers
     only ever read it.

## Viewing it locally before you deploy

Opening `index.html` directly from disk (`file://…`) will fail to load
`data/audits.json` in most browsers due to local file restrictions. Serve
the folder instead:

```bash
npx serve .
# or
python3 -m http.server 8000
```

Then open the printed `http://localhost:...` URL.

## Updating the data

- **Manually:** edit `data/audits.json` by hand (append a new object to
  `auditHistory`, following the schema), then validate before committing:

  ```bash
  node scripts/validate-data.js
  ```

- **Via a scheduled Claude Cowork audit:** see
  [`docs/WEEKLY_AUDIT_GUIDE.md`](docs/WEEKLY_AUDIT_GUIDE.md) for the exact
  process an automated weekly audit (using Claude in Chrome to check each
  scheduler, and the GitHub connector to commit the results) should
  follow, including the guardrails that keep the registry protected and
  the history append-only.

Either way: once the change is committed to `main` (directly, or via a
merged, CI-passing PR), GitHub Pages redeploys automatically and every
viewer sees the update — no separate publish step.
