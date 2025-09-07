# Web Archive Duplicator Removal

> Detect **exact** and **near-duplicate** web pages, **group** them, select a **canonical** page per group, and export a **decision-friendly JSON** + CSV table for review or downstream jobs.

---

## Why this exists

Large web crawls contain lots of repeated or lightly edited pages (mirrors, print views, tracking params, CMS duplicates). Grouping them:
- improves data quality for analytics and search,
- reduces storage/compute,
- gives reviewers a single **canonical** page per group.

This project **does not delete** pages. It **detects & groups** duplicates and reports the results.

---

## What you get (outputs)

- **JSON report** — near-duplicate pairs (with scores), groups, chosen canonicals, topic clusters, thresholds/metrics.
- **CSV table** — cleaned + annotated dataset (per-row duplicate labels, canonical flag, exact hash, etc.).
- **Notebook/Code** — reproducible pipeline with clear steps and tunable thresholds.

---

## How it works (pipeline)

Language filter (choose your language)
│
▼
Clean + normalize text (HTML/URLs/emails removed, lowercase, numbers marked)
│
▼
Lemmatize (spaCy, safe truncation for very long pages)
│
▼
TF-IDF vectors (unigrams + bigrams)
│
├── Stage-0 Exact dup: SHA-256(text_clean) → exact groups
│
└── Stage-1 Candidates: MinHash-LSH on 5-token shingles (Jaccard ≥ 0.85)
│
▼
Stage-2 Rerank: TF-IDF cosine ≥ 0.92 → strong pairs
│
▼
Similarity graph → connected components (duplicate groups)
│
▼
Canonical selection (longest cleaned text; HTTPS tie-break)
│
▼
JSON report + annotated CSV table




---

## Duplicate definitions (used here)

- **Exact duplicate** — identical **normalized** text; same `sha256(text_clean)`.
- **Near duplicate** — very high textual overlap; confirmed if **TF-IDF cosine ≥ 0.92**.
- **(Optional, future) Semantic duplicate** — same meaning with different wording/translation (embeddings).

---

## Techniques in plain English

- **Cleaning & normalization**: strips HTML, URLs, emails; normalizes unicode; lowercases; marks numbers; collapses whitespace.
- **Lemmatization (spaCy)**: reduces words to base forms (e.g., *running* → *run*) to cut inflection noise. Long pages are safely truncated (default 300k chars) to avoid memory issues.
- **TF-IDF**: converts text to a sparse vector that emphasizes terms frequent in a document but rare across the corpus (we use uni+bi-grams).
- **MinHash-LSH**: fast candidate generation by approximating Jaccard overlap over 5-token shingles; avoids O(N²) all-pairs.
- **Cosine rerank**: precise filter on TF-IDF vectors to keep only strong near-dups.
- **Graph grouping**: treat strong pairs as edges; connected components = duplicate groups.
- **Canonical**: pick the longest cleaned page; tie-break prefers HTTPS.

---

## Key parameters (tunable)

| Parameter | Default | Effect |
|---|---:|---|
| Shingle length `K` | 5 tokens | ↑K stricter; ↓K more recall |
| MinHash perms `NUM_PERM` | 128 | ↑perms better Jaccard estimate |
| LSH Jaccard `JACCARD_T` | 0.85 | candidate recall gate |
| Cosine rerank `NEAR_TFIDF_SIM` | 0.92 | final precision gate |
| TF-IDF vocab | 50k | cap feature space |
| TF-IDF `min_df` | 3 | drop ultra-rare terms |
| TF-IDF n-grams | (1,2) | unigrams + bigrams |
| spaCy truncation | 300k chars | stability on huge pages |

> Lower thresholds → more recall (more pairs to review).  
> Higher thresholds → fewer, cleaner matches.

---

## Output formats

### 1) JSON report (fields)

- **`meta`** — timestamp, counts, thresholds (`params`), TF-IDF vocab size.
- **`documents[]`** — per-doc info: `id`, `url`, `title`, `len_clean`, `exact_hash`, `exact_group_size`, `dup_group`, `dup_group_size`, `canonical_ix`, `is_canonical`, `cluster`.
- **`near_duplicate_pairs[]`** — `(i, j, tfidf_cosine, minhash_jaccard_est, url_i, title_i, url_j, title_j)`.
- **`near_duplicate_groups[]`** — group id, size, canonical doc, members, sample titles.
- **`exact_duplicate_groups[]`** — exact hash, size, members.
- **`topic_clusters[]`** — cluster id, size, top terms, sample docs (for reviewer context).

### 2) CSV table (columns summary)

`url, warc_date, content_type, title, text, len_text, languges, text_src, text_clean, title_clean, len_clean, tokens, lemmas, len_tokens, dup_group, dup_group_size, canonical_ix, is_canonical, exact_hash, exact_group_size`

> Filter `dup_group_size > 1` to see near-duplicates; use `is_canonical` to spot the representative.

---

## Quickstart

```bash
# Set up (example)
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m spacy download en_core_web_sm

# Run the notebook OR a CLI wrapper (if provided)
# Example CLI:
# python -m web_dedup.cli run \
#   --input data/sample.parquet \
#   --out reports/web_archive_dedup_report.json \
#   --config configs/default.yml


Example results (sample run)

Docs: 1,821

Near-dup pairs (kept): 1,845

Near-dup groups (size ≥ 2): 24

Exact-dup groups (size ≥ 2): 18

Topic clusters: 42

Sample derived from Common Crawl CC-MAIN-2025-33 (subset). Thresholds are configurable.

Review checklist (what a manager/teammate should look at)

JSON report: spot-check a few near-dup groups; confirm the canonical choice.

CSV: filter dup_group_size > 1 and skim titles/URLs.

Thresholds: are 0.85 (Jaccard) and 0.92 (cosine) acceptable for precision/recall?



Glossary

TF-IDF — weights terms high if frequent in a doc but rare across the corpus.

Shingles — overlapping n-grams (here, 5 tokens) describing local phrase content.

Jaccard similarity — set overlap = intersection / union.

MinHash — compact signature whose row-equality rate estimates Jaccard.

LSH (Locality-Sensitive Hashing) — buckets signatures so similar ones collide → fast candidate retrieval.

Connected component — group of docs linked by similarity edges (our near-dup group).

Canonical — the representative page we keep as the “main” one in a group.

License & attribution

Uses spaCy (en_core_web_sm) and scikit-learn; MinHash/LSH via datasketch.

Common Crawl data © respective site owners; see Common Crawl terms.




