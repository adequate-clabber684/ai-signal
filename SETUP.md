# Setup Guide

## 1. The one rule

**After every edit to `sources.yaml` — whether you made it manually or reviewed changes from a curation agent — run this before committing:**

```bash
python generate_opml.py
git add sources.yaml sources.opml
git commit -m "your message"
```

`sources.opml` is generated output. Never edit it by hand and never commit `sources.yaml` without it.

---

## 2. Day-to-day workflow

### Adding a source

1. Edit `sources.yaml` — add an entry following the schema
2. `python generate_opml.py`
3. Commit both files and push

### Updating an activity status

1. Edit `sources.yaml` — change `activity:` and update `last_checked:` to today's YYYY-MM
2. `python generate_opml.py`
3. Commit both files

### Reviewing a community PR

1. Check the new entry has all required fields
2. Verify the RSS feed URL works (paste into a browser or your reader)
3. If a landmark post is included, check the `why` is specific and useful
4. Merge — Pages redeploys automatically

### Responding to a staleness issue

The Monday workflow opens a GitHub issue when RSS feeds go quiet. For each flagged source:

1. Visit the source URL — check if it's actually dead or just RSS-broken
2. Edit `sources.yaml`: `activity: active` → `activity: slow` or `activity: archived`
3. `python generate_opml.py`, commit, close the issue

---

## 3. File reference

```
ai-signal/
├── sources.yaml                    ← THE data file — edit this
├── sources.opml                    ← Generated — do not edit by hand
├── index.html                      ← The website — SOURCES block is generated, do not edit it by hand
├── generate_opml.py                ← Run after every edit to sources.yaml (updates both sources.opml and index.html)
├── README.md                       ← Repo front page
├── contributing.md                 ← Contribution guide
├── SETUP.md                        ← This file
├── candidates.yaml                 ← Queue for source candidates (optional)
├── curate.py                       ← Optional RSS pre-fetcher (see §5)
├── tests/                          ← Tests for curate.py
└── .github/
    ├── labels.yml
    ├── ISSUE_TEMPLATE/
    │   ├── add-source.yml
    │   └── stale-source.yml
    ├── scripts/
    │   └── check_staleness.py
    └── workflows/
        ├── pages.yml               ← Deploy site on push to main
        └── staleness.yml          ← Weekly RSS freshness check
```

---

## 4. Staleness bot

The staleness workflow runs every Monday at 09:00 UTC. To test it manually:

1. Go to **Actions** tab in your repo
2. Click **Staleness Check** in the left list
3. Click **Run workflow** → **Run workflow**

It checks all RSS feeds and opens a GitHub issue if any are stale (labelled `staleness` + `automated`).

The workflow needs write permissions: **Settings → Actions → General → Workflow permissions → Read and write permissions**.

---

## 5. Optional: AI-assisted curation

Two Claude Code slash commands can do a full curation pass via web search — no RSS, no API key, just Claude Pro + Claude Code.

**This is optional.** Everything they do can be done manually by editing `sources.yaml` directly.

### Commands

**`/scan-sources`** — Quick activity check
Searches the web for each source, updates `activity` and `last_checked` in `sources.yaml` in-place.

**`/curate`** — Full curation pass
1. Checks all sources for activity (web search, not RSS)
2. Identifies landmark-worthy posts and adds them to `sources.yaml`
3. Evaluates any URLs in `candidates.yaml`
4. Discovers new source candidates
5. Appends new sources to `sources.yaml` marked `# PROPOSED`

### Review workflow

After the agent finishes, it prints a summary. Then:

```bash
git diff sources.yaml              # read what the agent changed
# edit sources.yaml if you want to adjust, remove, or accept PROPOSED entries
python generate_opml.py            # regenerate after any change you made
git add sources.yaml sources.opml
git commit -m "curate: YYYY-MM-DD"
git push
```

If you accept PROPOSED entries without changes, you still need to run `python generate_opml.py` — the agent already ran it, but if you edited anything afterward, run it again.

### Queuing candidate sources

To have `/curate` evaluate a specific URL, add it to `candidates.yaml` before running:

```yaml
candidates:
  - name: "Source Name"
    url: https://example.com
    rss: https://example.com/feed
```

Claude evaluates it against the criteria in `contributing.md` and either adds it to `sources.yaml` (marked `# PROPOSED`) or skips it. Remove entries from `candidates.yaml` after evaluation.

### Frequency

Run `/curate` monthly. `/scan-sources` any time activity statuses look stale.

---

## 6. Troubleshooting

**Pages shows a 404**
- Settings → Pages → Source must be "GitHub Actions" (not a branch)
- Check the Actions tab — the deploy workflow may have failed
- First deploy takes ~2 minutes

**Staleness workflow fails with permission error**
- Settings → Actions → General → Workflow permissions → Read and write permissions

**OPML doesn't download from the site**
- GitHub Pages serves `.opml` with the correct MIME type — should just work
- If not: right-click → Save Link As

**`generate_opml.py` fails**
```bash
pip install pyyaml
python generate_opml.py
```

**`curate.py` fails with "Missing deps"**
```bash
python -m pip install pyyaml feedparser
```

**curate.py shows 0 recent posts for an active source**
RSS parsing failure, not real inactivity. The `/curate` and `/scan-sources` commands use web search and will correctly identify the source as active. Ignore the curate.py output for those sources.
