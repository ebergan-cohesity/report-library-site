# Administrator Guide

Practical, day-to-day reference for running this report library once it's on
GitHub: sharing admin access, and how to edit, add, or retire reports.
For the deeper technical/historical record (the Oracle migration, local
Ruby/Jekyll environment setup and its Windows-specific quirks, and open
follow-up items) see [README.md](README.md) instead — this guide
deliberately doesn't repeat that material.

## Sharing administrative access

Since this repo will likely live under a Cohesity-owned GitHub organization
rather than a personal account, whether you personally can grant Admin
access to two colleagues depends on your own role in that org:

- **If you're an Organization Owner** (or the org's base permissions already
  allow it): go to the repo's **Settings → Collaborators and teams → Add
  people**, and set their role to **Admin** for full control (repo settings,
  branch protection, and — importantly — the ability to manage GitHub Pages
  configuration, which specifically requires Admin, not just Write, access).
- **If you're an Org Member** (the more common case for an individual
  contributor): you likely can't add collaborators or grant Admin yourself.
  You'll need to ask whoever administers Cohesity's GitHub org to either add
  the two colleagues directly, or — the more scalable, standard-practice
  approach for org-owned repos — create a **Team** scoped to this repo with
  Admin permission and add all three of you to it, so future access changes
  are a team-membership edit rather than a repo-by-repo one.
- **Check your own role first**: `github.com/orgs/<cohesity-org-name>/people`
  shows your role (Owner vs Member). That answers the question directly
  before you need to ask anyone else.

Either way, the two colleagues just need a GitHub account and Cohesity org
membership (if the org restricts access to members only) — nothing specific
to this repo's tooling is required to become an admin of it.

## Code owners and branch protection

Raised in an IT Security review: previously, anyone with write access could
push straight to the deploy branch and have it go live on the next build,
with no second-party review. Two controls now close that gap:

- **`.github/CODEOWNERS`** lists required reviewers per path — currently
  just `@ebergan-cohesity` for everything, including the content paths most
  worth a second look (`descriptions/`, `export/`). Add a colleague's GitHub
  handle to a line (space-separated for multiple owners on one path) once
  they're a confirmed collaborator/org member.
- **Branch protection** (Settings → Branches, on each repo) is what actually
  makes `CODEOWNERS` binding: "Require a pull request before merging" +
  "Require review from Code Owners" together block direct pushes to the
  deploy branch and require an owner's approval on the PR first. This is a
  repo *setting*, not a file, so it has to be configured separately per
  repo (personal and org) through the GitHub UI — see the security review
  doc for the exact steps.

Until both are set on a given repo, `CODEOWNERS` alone is just documentation
— it has no enforcement effect without the matching branch protection rule.

## How the site works, in one paragraph

Reports live as data, not as hand-written pages: legacy (Oracle-migrated)
reports' metadata sits in `export/reports.json`; locally-added reports each
carry their own `export/content/{id}/meta.json`. Optional detailed
explanations live in `descriptions/{id}.md`. Running
`python scripts/build_site_content.py` turns all of that into the actual
Jekyll site content (`_reports/`, `_products/`, `_categories/`, `reports/`,
`assets/search-index.json`) — idempotent, so it's always safe to rerun and
only touches whatever actually changed. **Any content edit requires this
rebuild step before it shows up anywhere** (local preview or the deployed
site) — editing `reports.json`, a `meta.json`, or a `descriptions/*.md` file
alone does nothing on its own.

GitHub Pages builds and deploys automatically on every push to the
configured branch, using a plain `jekyll build` — none of the local Windows
Ruby-environment fixes in the README (PATH, the Zscaler CA, the `tainted?`
shim) apply there; those exist purely to make local preview possible on
this machine.

## Editing an existing report

1. Find its metadata:
   - **Legacy report** (migrated from Oracle): an entry in
     `export/reports.json` — search that file for the report's `report_id`
     or `report_name`, and edit fields (`report_desc`, `problem_stmt`,
     `products`, `categories`, etc.) directly. It's one large JSON array;
     edit carefully and keep it valid JSON.
   - **Locally-added report**: its own
     `export/content/{report_id}/meta.json` — much simpler to edit since
     it's just that one report.
2. To replace the `.rtd` template, sample output, or thumbnail: overwrite
   the file in `export/content/{report_id}/` under the same filename.
3. To edit the detailed "How this report works" explanation: edit
   `descriptions/{report_id}.md` directly — plain markdown, no front
   matter, no special syntax.
4. To mark a report as officially Cohesity-supported, or set its "IT
   Analytics version(s) applicable": add or edit its entry in
   `export/overrides.json` — a small JSON object keyed by `report_id`, e.g.
   `{"1294": {"cohesity_supported": true, "ita_versions": "10.6+"}}`. Only
   reports that need something other than the default (not supported,
   versions unset) need an entry at all. Reports without an entry show as
   "not officially supported by Cohesity" and prompt an acknowledgment
   before download — this is the default for almost every report in the
   library.
5. Run `python scripts/build_site_content.py`, then preview locally
   (see README) before committing.

## Adding a new report

```
python scripts/new_report.py --rtd "My Report.rtd" --sample-html sample.html \
    --name "My Report" --desc "What it shows." \
    --problem-stmt "Why you'd run it." --author "you@example.com" \
    --products "Backup Manager / Veritas NetBackup" \
    --categories "Trending/Forecasting/Capacity Planning"
```
Use `--sample-image chart.png` instead of `--sample-html` if the sample
output is only a static image. Run `python scripts/new_report.py --help`
for the full option list and every valid `--products`/`--categories` value.

This allocates a `report_id` of 10000 or higher (legacy ids top out at
1297, so the ranges never collide), writes a self-contained
`export/content/{id}/` folder, and stubs out `descriptions/{id}.md`.
Then:
1. Run `python scripts/build_site_content.py`.
2. **Recommended**: ask Claude (or write it yourself) to read the new
   report's `.rtd`/sample output and fill in `descriptions/{id}.md` with a
   user-friendly explanation — this is what shows up under "How this
   report works" on the report page.
3. Preview locally, then commit and push.

## Retiring a report

Verified directly (added, built, confirmed removed, restored, confirmed
recovery — no manual cleanup needed either way):

- **Legacy report**: delete its object from the `export/reports.json`
  array (find it by `report_id`).
- **Locally-added report**: delete its `export/content/{report_id}/`
  folder, and `descriptions/{report_id}.md` if it has one.
- Either way, run `python scripts/build_site_content.py` — the generated
  report page and its download folder under `reports/{id}/` are removed
  automatically. No other reports are affected.

Once this is on GitHub, retiring a report through a normal commit (rather
than editing a live server directly) means the removed data is still fully
recoverable from git history if it turns out you need it back.

## Troubleshooting

- **Local preview looks completely unstyled** (no sidebar, everything
  rendering as a single flat, undifferentiated page): Jekyll has, on
  several occasions, silently served a completely different, wrong
  stylesheet instead of this site's own — a real, still not fully
  root-caused local-dev bug (see README's "Previewing locally" section for
  what's been ruled in/out). `scripts/serve.ps1` wipes `_site/`,
  `.jekyll-cache/`, and `.jekyll-metadata` before every start, which has
  reliably prevented it at startup — but it has also been seen once during
  an ordinary live-reload cycle on an already-running server, so don't
  fully trust the live preview after editing CSS/JS files either: if a page
  looks unstyled, stop the server and restart via `scripts/serve.ps1`
  rather than assuming the live reload picked it up correctly. This does
  not affect the deployed GitHub Pages site, which always builds fresh
  regardless.
- **An edit doesn't show up anywhere**: you forgot to rerun
  `python scripts/build_site_content.py` — see "How the site works" above.
