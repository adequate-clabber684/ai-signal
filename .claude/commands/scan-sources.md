# /scan-sources — AI Signal source activity checker

Check every source in `sources.yaml` for recent activity via web search, then
update `sources.yaml` directly with corrected `activity` and `last_checked`.

**Web search is the authority. RSS breaks silently — never rely on it alone.**

---

## Activity definitions

| Value | Means | Threshold |
|-------|-------|-----------|
| `active` | Posts regularly | At least once every ~6 weeks |
| `slow` | Infrequent but still publishing | Less than monthly, but posted in the last 18 months |
| `archived` | No longer updating | No posts in 18+ months — keep only if landmark posts exist |

## Activity rules

| Current status | Recent posts in last 90 days? | Action |
|----------------|-------------------------------|--------|
| `active` | Yes | No change |
| `active` | No | → `slow` |
| `slow` | Yes | → `active` |
| `slow` | No, last post < 18 months ago | No change |
| `slow` | No, last post > 18 months ago | → `archived` |
| `archived` | — | No change |

---

## Steps

### 1. Read sources

Read `sources.yaml` — get every source name, URL, and current `activity`.

### 2. Web-search each source for recent posts

Search: `site:{domain} {current year}` — e.g. `site:simonwillison.net 2026`
Fallback if no results: `"{source name}" blog post {current year}`

Determine:
- Published in the last 90 days? (yes / no)
- Approximate date of most recent post
- Any posts that look landmark-worthy? (title + URL — add them if yes)

### 3. Edit sources.yaml in-place

For every source where `activity` or `last_checked` changed:
- Update `activity:` field
- Set `last_checked:` to current YYYY-MM
- If you found a landmark post, append it to `landmark_posts:` too

### 4. Run generate_opml.py

```bash
python generate_opml.py
```

### 5. Print summary

```
Scanned N sources — YYYY-MM-DD
  ✓ confirmed active: [names]
  ↓ active → slow: [names + last post date]
  ↑ slow → active: [names + last post date]
  → archived: [names + last post date]
  ⚠ could not verify: [names — check manually]
  + landmark posts added: [source: post title]

Review:  git diff sources.yaml
Commit:  git add sources.yaml sources.opml && git commit -m "chore: activity scan YYYY-MM-DD"
```
