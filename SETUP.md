# Setup Guide

Everything you need to go from this zip to a live GitHub Pages site in under 15 minutes.

---

## 1. Create the GitHub repository

Go to https://github.com/new and fill in:

| Field | Value |
|-------|-------|
| **Repository name** | `ai-signal` |
| **Description** | A curated, structured reading layer for the AI field — filterable by audience, depth, and type. OPML included. |
| **Visibility** | Public (required for free GitHub Pages) |
| **Initialize** | Leave unchecked — you'll push the existing files |

Click **Create repository**.

---

## 2. Push the files

In your terminal, from inside the unzipped folder:

```bash
cd ai-signal

git init
git add .
git commit -m "feat: initial commit — 31 sources, OPML, Pages workflow"

git remote add origin https://github.com/YOUR_USERNAME/ai-signal.git
git branch -M main
git push -u origin main
```

Replace `YOUR_USERNAME` with your GitHub username.

---

## 3. Enable GitHub Pages

1. Go to your repo → **Settings** → **Pages** (left sidebar)
2. Under **Source**, select **GitHub Actions**
3. Click **Save**

The Pages deployment workflow (`.github/workflows/pages.yml`) will now trigger automatically on every push to `main`. Wait ~60 seconds for the first deploy.

Your site will be live at:
```
https://YOUR_USERNAME.github.io/ai-signal
```

---

## 4. Update the placeholder URLs

Do a find-and-replace across the repo for `yourusername` → your actual GitHub username. Files that need updating:

| File | What to change |
|------|---------------|
| `README.md` | Two occurrences of `yourusername/ai-signal` |
| `index.html` | Two GitHub links in the header and footer |
| `.github/scripts/check_staleness.py` | Fallback `REPO` value (line ~20) |

Then commit and push:
```bash
git add .
git commit -m "chore: update username placeholders"
git push
```

---

## 5. Apply GitHub labels

The repo uses custom labels for issue triage. Apply them with the GitHub CLI:

```bash
# Install gh CLI if you haven't: https://cli.github.com
gh auth login

# Apply labels (--force updates existing ones)
gh label create "add-source"      --color "0075ca" --description "PR or issue proposing a new source"        --force
gh label create "staleness"       --color "e4e669" --description "Source has gone quiet or feed is broken"    --force
gh label create "landmark-post"   --color "d93f0b" --description "Suggestion for a landmark post annotation"  --force
gh label create "remove-source"   --color "b60205" --description "Proposal to archive or remove a source"     --force
gh label create "data-quality"    --color "0e8a16" --description "Fix to metadata, tags, audience, or depth"  --force
gh label create "infrastructure"  --color "5319e7" --description "Changes to workflows, scripts, or site"     --force
gh label create "good-first-issue"--color "7057ff" --description "Good for newcomers to the project"          --force
gh label create "automated"       --color "cccccc" --description "Opened by the staleness bot"                --force
```

---

## 6. Set repo metadata on GitHub

On your repo homepage, click the **⚙ gear icon** next to "About" (top right of the code view) and fill in:

**Description:**
```
A curated, structured reading layer for the AI field — filterable by audience, depth, and type. OPML import included.
```

**Website:**
```
https://YOUR_USERNAME.github.io/ai-signal
```

**Topics (add all of these):**
```
ai  machine-learning  reading-list  curated-list  rss  opml  llm  ml-engineering  awesome-list  newsletter
```

---

## 7. Verify the staleness bot

The staleness workflow runs every Monday at 09:00 UTC. To test it manually:

1. Go to **Actions** tab in your repo
2. Click **Staleness Check** in the left list
3. Click **Run workflow** → **Run workflow**

It will check all 31 RSS feeds and open a GitHub issue if any are stale. The issue will be labelled `staleness` and `automated`.

The workflow needs permission to open issues. Verify under:
**Settings** → **Actions** → **General** → **Workflow permissions** → select **Read and write permissions**.

---

## 8. Optional: pin the site link in the README

The README already has a placeholder for the site URL. After confirming Pages is live, double-check the link works:
```
https://YOUR_USERNAME.github.io/ai-signal
```

---

## Day-to-day workflow

### Adding a source
1. Edit `sources.yaml` — add an entry following the schema
2. Run `python generate_opml.py` locally
3. Commit both `sources.yaml` and `sources.opml`
4. Push — Pages redeploys automatically

### Reviewing a community PR
1. Check the new entry has all required fields
2. Verify the RSS feed URL actually works (paste into a browser)
3. If a landmark post is included, check the `why` is specific and useful
4. Merge — Pages redeploys automatically

### Responding to a staleness issue
1. Open the issue (auto-generated every Monday if stale sources exist)
2. For each flagged source, visit the URL and check if it's actually dead
3. Edit `sources.yaml`: change `activity: active` → `activity: slow` or `activity: archived`
4. PR or direct push → close the issue

---

## File reference

```
ai-signal/
├── sources.yaml                    ← THE data file — edit this
├── sources.opml                    ← Generated — don't edit by hand
├── generate_opml.py                ← Run after editing sources.yaml
├── index.html                      ← The website (self-contained)
├── README.md                       ← Repo front page
├── contributing.md                 ← Contribution guide
├── SETUP.md                        ← This file
└── .github/
    ├── labels.yml                  ← Label definitions (reference)
    ├── ISSUE_TEMPLATE/
    │   ├── add-source.yml          ← Form for suggesting a source
    │   └── stale-source.yml        ← Form for reporting a dead source
    ├── scripts/
    │   └── check_staleness.py      ← Staleness checker (run by workflow)
    └── workflows/
        ├── pages.yml               ← Deploy site on push to main
        └── staleness.yml           ← Weekly RSS freshness check
```

---

## Troubleshooting

**Pages shows a 404**
- Check Settings → Pages → Source is set to "GitHub Actions" (not a branch)
- Check the Actions tab — the deploy workflow may have failed
- The first deploy takes ~2 minutes

**Staleness workflow fails with permission error**
- Settings → Actions → General → Workflow permissions → Read and write permissions

**OPML doesn't download from the site**
- GitHub Pages serves `.opml` files with the correct MIME type — should just work
- If not, try right-click → Save Link As

**generate_opml.py fails**
```bash
pip install pyyaml
python generate_opml.py
```
