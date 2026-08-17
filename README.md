# IT Analytics Report Library (GitHub Pages replacement)

Static replacement for reportlibrary.veritas.com. The original ~410 reports
were migrated out of the `RTD_REPORT` Oracle table (see
`scripts/export_from_oracle.py` for the queries/column notes) before that
Oracle environment was deprecated. `export/` is now the **committed source
of truth** going forward, not regenerable scratch data — `export/reports.json`
is a frozen snapshot of the legacy migration, and new reports are added
alongside it via `scripts/new_report.py` (below), each in its own
self-contained folder.

## Adding or correcting a report

**New report** — no Oracle, no manual JSON editing:
```
python scripts/new_report.py --rtd "My Report.rtd" --sample-html sample.html \
    --name "My Report" --desc "What it shows." \
    --problem-stmt "Why you'd run it." --author "you@example.com" \
    --products "Backup Manager / Veritas NetBackup" \
    --categories "Trending/Forecasting/Capacity Planning"
```
(Use `--sample-image chart.png` instead of `--sample-html` if the sample
output is only a static image — it gets wrapped in a bare HTML page so it
still displays through the same preview iframe as every other report.) Run
`python scripts/new_report.py --help` for the full option list, including the
valid `--products`/`--categories` values. This allocates a `report_id` of
10000 or higher (legacy Oracle ids top out at 1297, so the ranges never
collide), writes `export/content/{id}/` (the `.rtd`, sample, optional
thumbnail, and a `meta.json`), and stubs out `descriptions/{id}.md` for the
next step.

**Detailed explanations** — a user-friendly write-up of what a report does
and why, distinct from the short `--desc`/`--problem-stmt` fields, lives in
`descriptions/{report_id}.md`: plain markdown, no front matter, safe to
hand-edit. If present, it's rendered on the report page under "How this
report works." After adding a report (or for any of the ~410 legacy ones
that don't have one yet), ask Claude to read the `.rtd`/sample output and
draft `descriptions/{id}.md` — then review and edit it like any other
committed file.

**Correcting an existing report's metadata**: legacy reports' fields (title,
description, products, etc.) live in `export/reports.json`; locally-added
reports' live in their own `export/content/{id}/meta.json`. Either can be
hand-edited directly — there's no Oracle round-trip anymore.

After any of the above, regenerate the site:
```
python scripts/build_site_content.py
```
This is **idempotent** — it only rewrites files that actually changed (see
`write_if_changed`/`sync_dir` in `build_site_content.py`), so editing one
report's description and rerunning this touches one file, not the whole
site. Its summary line reports how many files actually changed vs. were
left alone.

## Legacy Oracle export (historical)

`scripts/export_from_oracle.py` is what originally produced
`export/reports.json` and `export/content/{id}/` for the ~410 migrated
reports, connecting either via a pre-opened SSH tunnel or one it manages
itself (see the script's docstring for both env-var setups). The source
Oracle environment is deprecated, so this script has no future runs planned
— kept only for reference/history. Do not re-run it against `export/`
expecting a refresh; there's nothing left to refresh it from.

## Previewing locally

```
bundle install
bundle exec jekyll serve
```
On Windows with a Ruby version newer than what `github-pages` was built
against, plain `bundle exec jekyll serve` may fail before it ever gets
here — see "Local Windows dev setup" below for the one-time fixes and the
`scripts/serve.ps1` wrapper that bakes them in.

Deliberately **not** using `--incremental`: it was the initial suspect
after Jekyll served a completely different, stale stylesheet (an old
placeholder theme's CSS instead of this site's own) instead of the real
one. Measured directly, it only shaved a few seconds off an already-slow
~140s local rebuild anyway (see Known Gaps below), so it wasn't worth
keeping regardless.

**This bug has recurred multiple times and is not fully root-caused.**
What's confirmed: a completely clean build (`_site/`, `.jekyll-cache/`,
`.jekyll-metadata` all absent beforehand) has been reliable every time it's
been tested, which is why `scripts/serve.ps1` now always deletes those
before starting. What's also confirmed, and not yet explained: it has
*also* happened once on an already-running server, mid-session, as the
result of an ordinary live regeneration cycle (editing and saving a CSS/JS
file while `jekyll serve` was watching) — so the bug isn't fully
contained to "restarting against leftover artifacts" the way it first
appeared to be. No stray `.scss` file, explicit `theme:` config, or
multiple concurrent Jekyll processes were found to explain it. Until this
is properly root-caused: **if a page ever looks completely unstyled (no
sidebar, everything as one flat column) - especially right after editing
`assets/css/style.css` or `assets/js/*.js` - stop the server and restart
it via `scripts/serve.ps1` rather than trusting the live auto-reload**, and
verify by comparing file sizes (`assets/css/style.css` vs
`_site/assets/css/style.css` should match exactly).

Once ready: for the very first push to a brand-new repo (before branch
protection exists), commit and push directly, then enable GitHub Pages
(Settings → Pages → Deploy from branch) pointing at the branch/root. For
every change after that, see ADMIN.md's "Submitting a change" — branch
protection is enabled on both repos today, so changes go through a pull
request rather than a direct push.

## Local Windows dev setup

`github-pages` pins Jekyll/Liquid to the old versions GitHub's own build
servers run, to keep local builds matching production. On a fresh Ruby
install (this was set up against Ruby 4.0.6 via RubyInstaller) three things
tend to bite before `bundle exec jekyll serve` will even start:

- **PATH not picked up in already-open shells** — RubyInstaller adds
  `C:\RubyXX-x64\bin` to the User PATH, but any shell/process started before
  the install won't see it until it re-reads the registry.
- **`bundle install` SSL certificate failures fetching gems** — if this
  network runs TLS-inspecting corporate proxy software (e.g. Zscaler), Ruby's
  OpenSSL uses its own CA bundle (`cacert.pem`), not the Windows trust store,
  so it doesn't automatically trust that proxy's root CA the way Windows/
  browsers do. Fix: export the proxy's root CA from
  `Cert:\LocalMachine\Root` and append it (PEM/base64 form) to Ruby's
  `cacert.pem`, then point `SSL_CERT_FILE` at that file.
- **`undefined method 'tainted?' for an instance of String'` on every page
  render** — Liquid 4.0.3 (pulled in by `github-pages`) still calls the
  `Object#tainted?`/`#taint`/`#untaint` API that Ruby removed in 3.2+.
  GitHub's actual production build servers run an internally-consistent
  older Ruby where this still exists, so it's a local-only problem. Fixed by
  `ruby_compat_shim.rb` at the repo root, which re-adds no-op versions of
  those methods. It's loaded via the `RUBYOPT` env var (`-r<path>`) rather
  than `_plugins/` or the Gemfile, because neither of those actually run
  under `bundle exec`/`github-pages`: `_plugins/` is silently disabled by
  Jekyll's safe mode (which `github-pages` forces locally to mirror GitHub's
  production behavior), and `bundle exec` reads the resolved `Gemfile.lock`
  at runtime rather than re-evaluating the Gemfile's Ruby source. `RUBYOPT`
  loads the shim at the interpreter level, before either of those come into
  play.

`scripts/serve.ps1` bundles all three fixes (PATH rebuild, `SSL_CERT_FILE`,
`RUBYOPT`), then runs `bundle exec jekyll serve --port 4000`. Run it from
anywhere:
```
powershell -File scripts\serve.ps1
```
Adjust the hardcoded `SSL_CERT_FILE` path inside it if your Ruby install
directory differs from `C:\Ruby40-x64`.

The `Gemfile` also pulls in `wdm` on Windows, so Jekyll watches for file
changes via native OS events instead of polling the whole tree — without it
you'll see "Please add wdm... to avoid polling for changes" and slower,
occasionally-stuttering rebuilds. This is unrelated to (and much safer than)
`--incremental` above - `wdm` only changes how file changes are *detected*,
not how the site is (re)built.

## Known gaps / follow-ups

- **A single-report edit takes ~100-140s for Jekyll's local dev server to
  pick up.** Measured directly: `build_site_content.py` itself is fast
  (idempotent, ~30s, touches only changed files) and `wdm` fires exactly one
  regeneration per real change instead of the old polling-storm behavior
  (multiple compounding full rebuilds per edit) — but Jekyll's own per-run
  scan across the ~7,000 static asset files under `reports/` dominates the
  remaining time regardless. `--incremental` claims to speed this up by
  skipping unchanged pages, but it's not used here (see above) since it
  twice corrupted the build instead. Fixing the underlying slowness would
  mean moving those static assets outside Jekyll's scanned source tree
  entirely (synced into `_site/reports/` by a separate script rather than
  Jekyll's own static-file pipeline) — a larger structural change, not
  attempted here since it's local-dev-only
  and doesn't affect the GitHub Pages production build.
- Product/category order in the sidebar is currently alphabetical rather than
  `DISPLAY_ORDER` from the old `RTD_PRODUCT` table — straightforward to wire
  in as a field on new/edited products if the original ordering matters.
- Search is a simple client-side JSON filter (`assets/js/search.js` +
  `assets/search-index.json`), not Pagefind. Fine at ~800 reports; worth
  upgrading to Pagefind (via a GitHub Actions build step) if the library
  grows much larger or full-text search inside descriptions is wanted.
- The old `RTD_PRODUCT_ID = 14` ("Drilldown Components / General") grouping
  is deliberately excluded from `scripts/build_site_content.py`'s `PRODUCTS`
  list — it's an internal grouping, not shown in the live site's public nav.
- `descriptions/`/`export/content/` (and everything else) now has a
  `.github/CODEOWNERS` file, and branch protection ("require PR" + "require
  review from Code Owners") is enabled on both repos — see ADMIN.md's "Code
  owners and branch protection" and "Submitting a change". That said, the
  catch-all path is the only one with a second owner (`@rgeller`,
  `@bseltz-cohesity`) so far; `descriptions/`/`export/content/` themselves
  are still owner-only, so PRs touching them still rely on an admin bypass
  rather than a real independent approval.
