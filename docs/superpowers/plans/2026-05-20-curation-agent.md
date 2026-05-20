# Curation Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a semi-automated curation agent that anyone with Claude Code can run locally to discover new AI sources and surface landmark-worthy posts from existing ones.

**Architecture:** A Python data-gatherer (`curate.py`) fetches recent RSS entries and writes a context file. `CLAUDE.md` gives Claude Code exact instructions for reasoning over that context, running web searches for new source discovery, and producing a `CURATION_REPORT.md` checklist for human review. No API key required — runs entirely within Claude Code (Claude Pro).

**Tech Stack:** Python 3.11+, pyyaml, feedparser, Claude Code (Claude Pro)

---

## File Map

| Action | File | Responsibility |
|--------|------|----------------|
| Create | `curate.py` | Fetch RSS feeds, write `_curate_context.json` |
| Create | `tests/test_curate.py` | Unit tests for curate.py |
| Create | `tests/__init__.py` | Make tests a package |
| Create | `CLAUDE.md` | Claude Code curation instructions |
| Create | `.gitignore` | Ignore `_curate_context.json` and `CURATION_REPORT.md` |
| Modify | `README.md` | Add Curation section |
| Modify | `SETUP.md` | Add curation workflow |
| Modify | `contributing.md` | Add curation agent section |

---

## Task 1: Project scaffolding

**Files:**
- Create: `.gitignore`
- Create: `tests/__init__.py`

- [ ] **Step 1: Create `.gitignore`**

```
# Curation agent — generated files
_curate_context.json
CURATION_REPORT.md
```

- [ ] **Step 2: Create `tests/__init__.py`**

Empty file — just create it:
```python
```

- [ ] **Step 3: Commit**

```bash
git add .gitignore tests/__init__.py
git commit -m "chore: add gitignore and tests package"
```

---

## Task 2: Build `curate.py` with TDD

**Files:**
- Create: `tests/test_curate.py`
- Create: `curate.py`

- [ ] **Step 1: Write the failing tests**

Create `tests/test_curate.py`:

```python
import json
import pytest
from datetime import datetime, timezone, timedelta
from pathlib import Path
from unittest.mock import patch, MagicMock


# ── load_sources ─────────────────────────────────────────────────────────────

def test_load_sources_returns_list(tmp_path):
    yaml_content = """
sources:
  - name: Test Blog
    url: https://test.com
    rss: https://test.com/feed.xml
    type: individual
    depth: deep
    audience: [researcher]
    tags: [test]
    activity: active
    last_checked: 2026-05
"""
    yaml_file = tmp_path / "sources.yaml"
    yaml_file.write_text(yaml_content)
    from curate import load_sources
    sources = load_sources(yaml_file)
    assert len(sources) == 1
    assert sources[0]["name"] == "Test Blog"
    assert sources[0]["rss"] == "https://test.com/feed.xml"


def test_load_sources_preserves_landmark_posts(tmp_path):
    yaml_content = """
sources:
  - name: Test Blog
    url: https://test.com
    rss: https://test.com/feed.xml
    type: individual
    depth: deep
    audience: [researcher]
    tags: [test]
    activity: active
    last_checked: 2026-05
    landmark_posts:
      - title: "A great post"
        url: https://test.com/great
        why: "Because it is great"
"""
    yaml_file = tmp_path / "sources.yaml"
    yaml_file.write_text(yaml_content)
    from curate import load_sources
    sources = load_sources(yaml_file)
    assert sources[0]["landmark_posts"][0]["title"] == "A great post"


# ── fetch_recent_entries ──────────────────────────────────────────────────────

def _make_mock_entry(title, url, days_ago, summary="Content"):
    """Helper: build a MagicMock feedparser entry."""
    entry = MagicMock()
    entry.title = title
    entry.link = url
    pub = datetime.now(timezone.utc) - timedelta(days=days_ago)
    entry.published_parsed = (pub.year, pub.month, pub.day, 0, 0, 0, 0, 0, 0)
    entry.updated_parsed = None
    entry.summary = summary
    return entry


def test_fetch_recent_entries_includes_recent_post():
    mock_feed = MagicMock()
    mock_feed.entries = [_make_mock_entry("New Post", "https://test.com/new", days_ago=5)]
    with patch("feedparser.parse", return_value=mock_feed):
        from curate import fetch_recent_entries
        since = datetime.now(timezone.utc) - timedelta(days=90)
        result = fetch_recent_entries("https://test.com/feed.xml", since)
    assert len(result) == 1
    assert result[0]["title"] == "New Post"
    assert result[0]["url"] == "https://test.com/new"


def test_fetch_recent_entries_excludes_old_post():
    mock_feed = MagicMock()
    mock_feed.entries = [_make_mock_entry("Old Post", "https://test.com/old", days_ago=120)]
    with patch("feedparser.parse", return_value=mock_feed):
        from curate import fetch_recent_entries
        since = datetime.now(timezone.utc) - timedelta(days=90)
        result = fetch_recent_entries("https://test.com/feed.xml", since)
    assert result == []


def test_fetch_recent_entries_handles_network_error():
    with patch("feedparser.parse", side_effect=Exception("network error")):
        from curate import fetch_recent_entries
        since = datetime.now(timezone.utc) - timedelta(days=90)
        result = fetch_recent_entries("https://test.com/feed.xml", since)
    assert result == []


def test_fetch_recent_entries_caps_at_twenty():
    entries = [_make_mock_entry(f"Post {i}", f"https://t.com/{i}", days_ago=1) for i in range(30)]
    mock_feed = MagicMock()
    mock_feed.entries = entries
    with patch("feedparser.parse", return_value=mock_feed):
        from curate import fetch_recent_entries
        since = datetime.now(timezone.utc) - timedelta(days=90)
        result = fetch_recent_entries("https://test.com/feed.xml", since)
    assert len(result) <= 20


def test_fetch_recent_entries_truncates_summary():
    long_summary = "x" * 500
    mock_feed = MagicMock()
    mock_feed.entries = [_make_mock_entry("Post", "https://t.com/p", days_ago=1, summary=long_summary)]
    with patch("feedparser.parse", return_value=mock_feed):
        from curate import fetch_recent_entries
        since = datetime.now(timezone.utc) - timedelta(days=90)
        result = fetch_recent_entries("https://test.com/feed.xml", since)
    assert len(result[0]["summary"]) <= 300


# ── main / output ─────────────────────────────────────────────────────────────

def test_main_writes_valid_json(tmp_path, monkeypatch):
    yaml_content = """
sources:
  - name: Test Blog
    url: https://test.com
    rss: https://test.com/feed.xml
    type: individual
    depth: deep
    audience: [researcher]
    tags: [test]
    activity: active
    last_checked: 2026-05
"""
    (tmp_path / "sources.yaml").write_text(yaml_content)
    output_file = tmp_path / "_curate_context.json"

    monkeypatch.chdir(tmp_path)

    mock_feed = MagicMock()
    mock_feed.entries = []
    with patch("feedparser.parse", return_value=mock_feed):
        import importlib, curate
        importlib.reload(curate)
        curate.OUTPUT_FILE = output_file
        curate.main()

    data = json.loads(output_file.read_text())
    assert "sources" in data
    assert data["sources"][0]["name"] == "Test Blog"
    assert "generated_at" in data
    assert "lookback_days" in data
```

- [ ] **Step 2: Run tests — confirm they all fail**

```bash
cd "D:\_blog\ai-signal"
python -m pip install pyyaml feedparser pytest --quiet
python -m pytest tests/test_curate.py -v
```

Expected: `ModuleNotFoundError: No module named 'curate'` or similar — all tests fail.

- [ ] **Step 3: Implement `curate.py`**

Create `curate.py`:

```python
#!/usr/bin/env python3
"""
curate.py — Gather RSS data for the AI Signal curation agent.

Usage:
    python -m pip install pyyaml feedparser
    python curate.py

Outputs _curate_context.json (gitignored), then open Claude Code and say:
    run curation
"""

import json
import sys
from datetime import datetime, timezone, timedelta
from pathlib import Path

try:
    import feedparser
    import yaml
except ImportError:
    print("Missing deps. Run: python -m pip install pyyaml feedparser")
    sys.exit(1)

LOOKBACK_DAYS = 90
OUTPUT_FILE = Path("_curate_context.json")


def load_sources(path: Path) -> list[dict]:
    with path.open() as f:
        return yaml.safe_load(f)["sources"]


def fetch_recent_entries(rss_url: str, since: datetime) -> list[dict]:
    try:
        feed = feedparser.parse(rss_url)
        entries = []
        for entry in feed.entries[:20]:
            for field in ("published_parsed", "updated_parsed"):
                t = getattr(entry, field, None)
                if t:
                    dt = datetime(*t[:6], tzinfo=timezone.utc)
                    if dt >= since:
                        entries.append({
                            "title": getattr(entry, "title", ""),
                            "url": getattr(entry, "link", ""),
                            "date": dt.strftime("%Y-%m-%d"),
                            "summary": getattr(entry, "summary", "")[:300],
                        })
                    break
        return entries
    except Exception:
        return []


def main():
    sources = load_sources(Path("sources.yaml"))
    since = datetime.now(timezone.utc) - timedelta(days=LOOKBACK_DAYS)

    context = {
        "generated_at": datetime.now(timezone.utc).isoformat(),
        "lookback_days": LOOKBACK_DAYS,
        "sources": [],
    }

    for s in sources:
        print(f"Fetching: {s['name']} ...", end=" ", flush=True)
        rss = s.get("rss")
        recent = []
        if rss:
            recent = fetch_recent_entries(rss, since)
            print(f"{len(recent)} recent posts")
        else:
            print("no RSS")

        context["sources"].append({
            "name": s["name"],
            "url": s.get("url", ""),
            "rss": rss or "",
            "type": s.get("type", ""),
            "depth": s.get("depth", ""),
            "audience": s.get("audience", []),
            "tags": s.get("tags", []),
            "activity": s.get("activity", ""),
            "last_checked": s.get("last_checked", ""),
            "landmark_posts": s.get("landmark_posts", []),
            "recent_posts": recent,
        })

    OUTPUT_FILE.write_text(json.dumps(context, indent=2, ensure_ascii=False))
    total = len(context["sources"])
    print(f"\nContext written to {OUTPUT_FILE} ({total} sources)")
    print("Now open Claude Code in this directory and say: run curation")


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run tests — confirm they all pass**

```bash
python -m pytest tests/test_curate.py -v
```

Expected output:
```
tests/test_curate.py::test_load_sources_returns_list PASSED
tests/test_curate.py::test_load_sources_preserves_landmark_posts PASSED
tests/test_curate.py::test_fetch_recent_entries_includes_recent_post PASSED
tests/test_curate.py::test_fetch_recent_entries_excludes_old_post PASSED
tests/test_curate.py::test_fetch_recent_entries_handles_network_error PASSED
tests/test_curate.py::test_fetch_recent_entries_caps_at_twenty PASSED
tests/test_curate.py::test_fetch_recent_entries_truncates_summary PASSED
tests/test_curate.py::test_main_writes_valid_json PASSED
8 passed
```

- [ ] **Step 5: Smoke-test against real data**

```bash
python curate.py
```

Expected: progress lines per source, then `Context written to _curate_context.json (31 sources)`. Spot-check the file:

```bash
python -c "import json; d=json.load(open('_curate_context.json')); print(d['sources'][0]['name'], '-', len(d['sources'][0]['recent_posts']), 'posts')"
```

- [ ] **Step 6: Commit**

```bash
git add curate.py tests/test_curate.py tests/__init__.py
git commit -m "feat: add curate.py — RSS data gatherer for curation agent"
```

---

## Task 3: Create `CLAUDE.md` — curation instructions

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: Create `CLAUDE.md`**

```markdown
# AI Signal — Claude Code Instructions

## Project overview

AI Signal is a curated reading list for the AI field. The canonical data lives in
`sources.yaml` (31 sources, 6 categories). Everything else is generated from it:

- `generate_opml.py` → `sources.opml` (RSS reader import file)
- `curate.py` → `_curate_context.json` (input for the curation agent)
- `index.html` — static filterable site, deployed to GitHub Pages on push

**After any change to `sources.yaml`:** run `python generate_opml.py` and commit
both `sources.yaml` and `sources.opml` together.

---

## Curation agent

When asked to **"run curation"**, **"curate"**, **"check for new sources"**, or
**"find new sources"**, follow these steps exactly.

### Step 1 — Gather RSS data

Check whether `_curate_context.json` exists and contains today's date in
`generated_at`. If not (or if it's stale), run:

```bash
python -m pip install pyyaml feedparser --quiet
python curate.py
```

This takes ~1 minute. It fetches the last 90 days of posts from all 31 RSS feeds
and writes them to `_curate_context.json`.

### Step 2 — Read the context

Read `_curate_context.json` in full. For each source, note:

- **`recent_posts`** count: 0 posts in 90 days on an `active` source → candidate
  for `slow`; 0 posts on a `slow` source with `last_checked` > 6 months ago →
  candidate for `archived`
- **Post titles**: scan for titles that suggest synthesis, surveys, deep dives,
  or naming a new concept — not product announcements or release notes

### Step 3 — Identify landmark post candidates (existing sources)

A post is landmark-worthy if it meets all three:
1. Covers a concept or technique not already in that source's `landmark_posts`
2. Would be useful to an ML engineer reading it 12 months from now (not news)
3. Demonstrates unusual depth, breadth, or clarity for the topic

Flag the post with its URL and a one-sentence `why`.

### Step 4 — Discover new sources

Run these web searches to find source candidates not yet in `sources.yaml`:

```
"AI blog" OR "ML blog" 2025 -site:medium.com recommendations
"best AI newsletters 2025" OR "top ML blogs 2025"
site:news.ycombinator.com "who writes" AI blog OR newsletter 2025
"AI research blog" recommendations 2025
new AI researchers blog substack 2025
```

Also consider: any author cited in recent landmark posts who publishes regularly
but doesn't have their own entry in `sources.yaml`.

For each candidate, verify all of the following before including it:
- Has a working RSS feed URL
- Published at least 3 posts in the last 90 days OR has landmark-quality archive
- Passes the contributing.md "signal, not noise" test (specific point of view,
  not a tutorial farm, not primarily marketing)
- Not already in `sources.yaml`
- Not paywalled

### Step 5 — Write `CURATION_REPORT.md`

Write the report using this exact format. Omit any section where there is nothing
to report (no candidates, no changes needed).

---

```markdown
# Curation Report — YYYY-MM-DD

_Generated by the AI Signal curation agent. Review each item and apply changes
to `sources.yaml` as appropriate. Run `python generate_opml.py` after applying,
then commit `sources.yaml` and `sources.opml` together._

---

## New Source Candidates

- [ ] **[Source Name]** — `type: X` · `depth: X` · `audience: [X, Y]`
  **URL:** https://...
  **RSS:** https://...feed.xml
  **Why it qualifies:** [1–2 sentences against the contributing.md criteria]
  **Recent posts (sample):**
  - "Post title one" (YYYY-MM-DD)
  - "Post title two" (YYYY-MM-DD)
  **Suggested tags:** `[tag1, tag2, tag3]`
  **Suggested `sources.yaml` entry:**
  ```yaml
  - name: "Source Name"
    url: https://...
    rss: https://...feed.xml
    type: X
    depth: X
    audience: [X, Y]
    tags: [tag1, tag2, tag3]
    activity: active
    last_checked: YYYY-MM
  ```

---

## New Landmark Posts (existing sources)

- [ ] **[Source Name]** → "[Post Title]"
  **URL:** https://...
  **Date:** YYYY-MM-DD
  **Why landmark-worthy:** [1 sentence]
  **Suggested addition to `sources.yaml`:**
  ```yaml
  landmark_posts:
    - title: "Post Title"
      url: https://...
      why: "One sentence on why this specific post matters"
  ```

---

## Activity Status Changes

- [ ] **[Source Name]**: `active` → `slow`
  **Reason:** 0 posts in last 90 days (last post visible: YYYY-MM-DD or unknown)
  **Action:** Change `activity: active` to `activity: slow` in `sources.yaml`

---

## Notes

[Any other observations — broken feed URLs, sources whose scope has drifted,
duplicate sources, etc.]
```

---

## Applying the report

After reviewing `CURATION_REPORT.md`:

1. Edit `sources.yaml` for each checked item
2. Run `python generate_opml.py`
3. Commit: `git add sources.yaml sources.opml && git commit -m "curate: YYYY-MM-DD curation pass"`
4. Push to main — GitHub Pages redeploys automatically
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "feat: add CLAUDE.md with curation agent instructions"
```

---

## Task 4: Update documentation

**Files:**
- Modify: `README.md`
- Modify: `SETUP.md`
- Modify: `contributing.md`

- [ ] **Step 1: Add Curation section to `README.md`**

Find the `## Automation` section in `README.md` (line ~88). After the closing
paragraph of that section, add a new `## Curation` section:

```markdown
## Curation

A local curation agent helps discover new sources and surface landmark posts from
existing ones. It runs entirely within Claude Code — no API key needed.

**Requirements:** Claude Code (Claude Pro), Python 3.11+

```bash
python -m pip install pyyaml feedparser
python curate.py          # ~1 min — fetches recent RSS data from all sources
```

Then open Claude Code in this directory and say: **run curation**

Claude reads the gathered RSS data, searches the web for new source candidates,
and writes `CURATION_REPORT.md` — a structured checklist of proposals. You review
it, apply the items you agree with to `sources.yaml`, then:

```bash
python generate_opml.py
git add sources.yaml sources.opml
git commit -m "curate: YYYY-MM-DD curation pass"
```

See `SETUP.md` for the full curation workflow.
```

- [ ] **Step 2: Add curation section to `SETUP.md`**

Append a new section after the existing content in `SETUP.md`:

```markdown
---

## 4. Running the curation agent

The curation agent discovers new AI sources and surfaces landmark-worthy posts
from existing ones. Anyone with Claude Code (Claude Pro) can run it.

### Prerequisites

```bash
python -m pip install pyyaml feedparser
```

### Workflow

**1. Gather RSS data (~1 min)**

```bash
python curate.py
```

This reads all 31 sources from `sources.yaml`, fetches the last 90 days of posts
from each RSS feed, and writes `_curate_context.json`. This file is gitignored.

**2. Run the curation agent**

Open Claude Code in this directory and say:

> run curation

Claude will:
- Read the RSS context
- Identify posts on existing sources that look landmark-worthy
- Search the web for new AI sources that meet the curation criteria
- Flag any sources with no recent posts (activity status candidates)

**3. Review `CURATION_REPORT.md`**

Claude writes a structured checklist. Each item is a checkbox. Review the
proposals and tick the ones you want to apply.

**4. Apply changes and commit**

For each checked item, edit `sources.yaml` directly. Then:

```bash
python generate_opml.py
git add sources.yaml sources.opml
git commit -m "curate: YYYY-MM-DD curation pass"
git push
```

GitHub Pages redeploys automatically within ~2 minutes.

### Frequency

Run the curation agent monthly, or whenever you want to refresh the list.
The existing staleness workflow (GitHub Actions, every Monday) already handles
dead feed detection — the curation agent goes deeper: quality, discovery,
landmark posts.

### Troubleshooting

**`curate.py` exits with "Missing deps"**
```bash
python -m pip install pyyaml feedparser
```

**Claude says `_curate_context.json` not found**
Run `python curate.py` first, then ask Claude to run curation again.

**A feed returns 0 recent posts but the source is clearly active**
Some feeds use non-standard date fields. Claude will flag this in the Notes
section of the report — verify manually.
```

- [ ] **Step 3: Add curation section to `contributing.md`**

Append after the `## License` section in `contributing.md`:

```markdown
---

## Using the curation agent

The curation agent is the recommended way to discover new sources to add and to
audit existing sources for landmark post candidates.

See `SETUP.md §4` for the full workflow. Short version:

```bash
python curate.py           # gather RSS data
# open Claude Code → say: run curation
# review CURATION_REPORT.md → apply to sources.yaml
python generate_opml.py
git add sources.yaml sources.opml && git commit -m "curate: YYYY-MM-DD"
```

When submitting a PR for sources found via the curation agent, note that in your
PR description — it helps reviewers understand the provenance.
```

- [ ] **Step 4: Commit documentation updates**

```bash
git add README.md SETUP.md contributing.md
git commit -m "docs: document curation agent in README, SETUP, contributing"
```

---

## Task 5: End-to-end verification

- [ ] **Step 1: Run the full test suite**

```bash
python -m pytest tests/ -v
```

Expected: 8 passed, 0 failed.

- [ ] **Step 2: Run `curate.py` against real data**

```bash
python curate.py
```

Verify `_curate_context.json` exists and contains at least 5 sources with
non-empty `recent_posts`.

- [ ] **Step 3: Verify `.gitignore` is working**

```bash
git status
```

`_curate_context.json` should NOT appear in the untracked files list.

- [ ] **Step 4: Final commit check**

```bash
git log --oneline -5
```

Expected — four commits since start of this work:
```
docs: document curation agent in README, SETUP, contributing
feat: add CLAUDE.md with curation agent instructions
feat: add curate.py — RSS data gatherer for curation agent
chore: add gitignore and tests package
```
