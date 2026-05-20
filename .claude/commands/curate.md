# /curate — AI Signal full curation pass

Full curation: verify existing sources, discover new ones, surface landmark posts.
All changes go directly into `sources.yaml`. Review with `git diff`, then run
`python generate_opml.py` and commit.

**Do NOT write a report file. Edit `sources.yaml` directly.**

---

## Vocabulary — use these definitions consistently

### Source types
| Value | Use when |
|-------|----------|
| `lab` | Official research blog of an AI lab (Anthropic, OpenAI, DeepMind, Meta AI, BAIR, etc.) |
| `individual` | A person's personal blog or newsletter — single author, strong POV |
| `systems` | Infrastructure, inference, GPU serving, ML platform engineering |
| `paper-explainer` | Newsletters or blogs that translate research papers for practitioners |
| `applied` | Company engineering blog primarily about building *with* AI (tools, evals, production) |
| `news` | Weekly digest or news aggregator — broad coverage, not deep technical |

### Depth
| Value | Means |
|-------|-------|
| `deep` | Assumes strong ML background. Follows math notation, knows KL divergence, attention internals. Posts reward focused reading time. |
| `medium` | Assumes software engineering competence. Introduces ML concepts without drowning in them. Most working ML engineers can follow without a textbook. |
| `shallow` | Good for orientation. Summaries, digests, news. Low time commitment. |

### Audience
| Value | Means |
|-------|-------|
| `researcher` | PhD-level ML background expected. Reads papers, implements from scratch. |
| `ml-engineer` | Strong engineering + solid ML fundamentals. Trains and deploys models. |
| `builder` | Software engineer building AI products. Uses APIs and frameworks, not training from scratch. |
| `curious` | Technical but not ML-specialist. Wants to stay oriented, not go deep. |

Use multiple audiences only when genuinely appropriate. A `researcher`-only source should not also get `builder` to seem inclusive.

### Activity
| Value | Means | Threshold |
|-------|-------|-----------|
| `active` | Posts regularly | At least once every ~6 weeks |
| `slow` | Infrequent but still publishing | Less than monthly, but posted in last 18 months |
| `archived` | No longer updating | No posts in 18+ months — keep only if landmark posts exist |

### Landmark posts
A landmark post is one that meets ALL three:
1. Covers a concept or technique not already in that source's `landmark_posts` list
2. Will be useful to an ML engineer reading it 12 months from now — not news, not a release note, not a product announcement
3. Demonstrates unusual depth, breadth, or clarity — something you'd recommend to a colleague by name

The `why` field must be specific: not "great post about RAG" but "the clearest explanation of why naive RAG fails at scale and exactly which retrieval strategies fix it."

---

## Step 1 — Activity scan (web search, not RSS)

Read `sources.yaml`. For each source, run: `site:{domain} {current year}`
Example: `site:simonwillison.net 2026`

If that returns nothing: `"{source name}" blog post {current year}`.

Determine: has the source published in the last 90 days? What is the most recent post date?

Apply the activity rules:

| Current | Recent posts (last 90 days)? | Action |
|---------|------------------------------|--------|
| `active` | Yes | No change |
| `active` | No | → `slow` |
| `slow` | Yes | → `active` |
| `slow` | No, last post < 18 months ago | No change |
| `slow` | No, last post > 18 months ago | → `archived` |
| `archived` | — | No change |

Update `last_checked` to current YYYY-MM for every source you verify.
Edit `activity` and `last_checked` in-place — do not change other fields unless you also find a landmark post.

---

## Step 2 — Landmark post detection on existing sources

While scanning each source, check whether any recent posts are landmark-worthy (see definition above). Add qualifying posts to that source's `landmark_posts` in `sources.yaml`:

```yaml
landmark_posts:
  - title: "Post Title"
    url: https://...
    why: "Specific one sentence — what this teaches and why it matters, not 'great post about X'"
```

---

## Step 3 — Evaluate candidates.yaml

If `candidates.yaml` exists and has entries, evaluate each one using the five-point check below. Do this before running discovery searches so you don't duplicate work.

---

## Step 4 — Discover new sources

This step must find sources NOT currently in `sources.yaml`. Run all of these searches:

```
best AI ML technical blogs 2026 -site:medium.com -site:kdnuggets.com
top AI newsletters 2026 substack researchers practitioners
site:news.ycombinator.com "AI blog" OR "ML blog" 2026
new AI researcher blog active 2026
mechanistic interpretability blog 2026
AI inference serving systems blog 2026
AI safety alignment blog active 2026
LLM fine-tuning quantization blog 2026
applied AI engineering blog production 2026
```

Also consider: authors cited repeatedly in recent landmark posts from existing sources who publish their own blog not yet in the list.

**Explicitly excluded categories** — do not propose these regardless of quality:
- Paywalled sources (readers must be able to access it freely)
- Tutorial farms (sites that exist to rank on Google, not share genuine knowledge)
- Primarily marketing or product announcement blogs
- Podcast-only sources (no written content, or transcripts only)
- Generic aggregators with no editorial judgement
- Sources with no RSS feed (can't be imported into an RSS reader)
- Dead sources (no posts in 18+ months) unless landmark posts justify archiving

**Five-point check for every candidate:**
1. Has a working RSS feed URL?
2. Published at least 3 posts in the last 90 days (or landmark-quality archive)?
3. Has a specific editorial POV — not a link dump, not generic?
4. Not already in `sources.yaml`?
5. Not paywalled?

Only propose sources that pass all five.

---

## Step 5 — Edit sources.yaml directly

### Existing sources — edit in-place
Change `activity:`, update `last_checked:`, append to `landmark_posts:`. No markers needed for changes to existing entries.

### New source candidates — append with PROPOSED marker
Add at the bottom of the correct section, with a `# PROPOSED` comment:

```yaml
  # PROPOSED — review before committing
  - name: "Source Name"
    url: https://...
    rss: https://...
    type: X
    depth: X
    audience: [X, Y]
    tags: [tag1, tag2, tag3]
    activity: active
    last_checked: 2026-05
```

After evaluating candidates.yaml entries (pass or fail), remove them from candidates.yaml.

---

## Step 6 — Run generate_opml.py

```bash
python generate_opml.py
```

---

## Step 7 — Print summary

```
Curation complete — YYYY-MM-DD
  Sources scanned: N
  Activity updated: N (list names + change)
  Landmark posts added: N (source name + post title)
  New sources proposed: N (list names, marked # PROPOSED)
  Candidates evaluated: N (pass/fail breakdown)

Review:  git diff sources.yaml
Commit:  git add sources.yaml sources.opml && git commit -m "curate: YYYY-MM-DD"
Push:    git push
```
