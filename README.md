# Redrob Hackathon — Intelligent Candidate Discovery & Ranking

**Team:** masterfighter  
**Participant:** Swayam Chatterjee  
**Challenge:** Data and AI Challenge: Intelligent Candidate Discovery  

---

## What this does

Ranks 100,000 candidate profiles against a Senior AI Engineer job description, producing a top-100 submission CSV. The full ranking pipeline runs in **~70 seconds on a 2-core CPU** with no GPU, no network calls, and no third-party ML libraries — only Python stdlib.

---

## Repository structure

```
├── phase2_ranking_fixed.ipynb   # Main scoring pipeline (run this)
├── jd_spec_fixed.json           # Structured JD spec — Phase 1 output
├── company_founding_years.json  # 63-company founding-year table for honeypot detection
├── candidate_schema.json        # Schema reference for the candidate pool
├── jd-spec-creation.ipynb       # Phase 1 notebook: how jd_spec_fixed.json was produced
├── submission_metadata.yaml     # Full methodology, compute env, AI tool declaration
└── README.md
```

---

## How to reproduce

### Prerequisites

- Python 3.9+ (tested on 3.11)
- No third-party packages required — the scoring pipeline uses Python stdlib only (`json`, `csv`, `re`, `math`, `gzip`, `concurrent.futures`)
- `candidates.jsonl` (or `candidates.jsonl.gz`) from the hackathon bundle — **not included in this repo**

### Option 1 — Google Colab (recommended)

1. Open `phase2_ranking_fixed.ipynb` in Colab
2. Upload the following files to `/content/`:
   - `candidates.jsonl` (or `candidates.jsonl.gz`) — the full 100K candidate pool
   - `jd_spec_fixed.json`
   - `company_founding_years.json`
3. Run all cells in order (`Runtime → Run all`)
4. `submission.csv` will be written to `/content/submission.csv`

> **Important:** Upload the complete `candidates.jsonl` before running. The notebook includes a file-completeness check in Cell 2 that raises a hard error if the file is not exactly 100,000 lines — this guards against silent truncation from slow/interrupted uploads to Colab's ephemeral `/content/` storage.

### Option 2 — Local machine

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPO
cd YOUR_REPO

# Place candidates.jsonl in the same directory, then:
jupyter nbconvert --to notebook --execute phase2_ranking_fixed.ipynb \
  --ExecutePreprocessor.timeout=600 \
  --output phase2_ranking_executed.ipynb
```

The output CSV will be at `/content/submission.csv` (Colab path hardcoded in the notebook — change `OUTPUT_CSV` in Cell 2 if running locally).

**Expected runtime:** 60–80 seconds on a 2-core CPU.

---

## Architecture overview

The pipeline has two phases:

### Phase 1 — Offline, one-time (not part of the 5-minute budget)

`jd-spec-creation.ipynb` uses an LLM to extract a machine-readable scoring spec from the prose job description. Output: `jd_spec_fixed.json`. This runs once and is committed to the repo — the scoring notebook reads it as a static file.

Also produced: `company_founding_years.json`, a manually verified 63-company founding-year reference table used for honeypot detection.

### Phase 2 — Online, CPU-only, no network

`phase2_ranking_fixed.ipynb` applies the spec to all 100,000 candidates:

| Cell | What it does |
|------|-------------|
| Cell 2 | Imports, path config, file-completeness guard |
| Cell 4 | Loads `jd_spec_fixed.json` → builds `REQ_LOOKUP`, `BONUS_LOOKUP`, `FOUNDING_YEARS`, `DIM_W` |
| Cell 6 | All scoring functions: honeypot detection, hard/soft disqualifiers, 7 dimension scorers, `career_relevance_multiplier`, `score_candidate` |
| Cell 8 | Composite scorer + per-candidate reasoning generator |
| Cell 10 | Streaming JSONL loader + `ProcessPoolExecutor` for parallel scoring |
| Cell 12 | CSV writer with tie-break enforcement (non-increasing score; candidate_id ascending on ties) |
| Cell 14–16 | Inline format validation + honeypot audit of top 100 |

### Scoring formula

```
final_score = weighted_sum × soft_disqualifier_multiplier × career_relevance_multiplier
```

**Dimension weights** (from `jd_spec_fixed.json`):

| Dimension | Weight |
|-----------|--------|
| Required signals | 35% |
| Experience fit | 25% |
| Bonus signals | 9% |
| Location / logistics | 10% |
| Availability / freshness | 10% |
| Hireability signals | 8% |
| Verification signals | 3% |

### Three-tier evidence model

Keyword matching distinguishes how strongly a signal is evidenced:

- **Full weight** — keyword found in `career_history` descriptions/titles (candidate narrated doing the work)
- **0.45× weight** — keyword only in `skills[]`, discounted by proficiency level and `duration_months`
- **0.25× weight** — keyword only in headline/summary, no career corroboration

This directly addresses the JD's keyword-stuffing trap: a candidate with "FAISS" listed as a skill earns 0.45× credit, not full credit.

### Honeypot detection

Two patterns, both verified empirically against the full 100K dataset:

1. **Skill duration mismatch** — `proficiency = "expert"` with `duration_months < 12` across 3+ skills simultaneously
2. **Predates company founding** — `career_history` start date is 4+ years before the company's known founding year

The 4-year threshold is empirically tuned: an unthresholded check flagged 281 candidates (3× the expected ~80 honeypots). Gap-distribution analysis showed 1–2 year gaps are synthetic-data noise; only 4+ year gaps represent genuinely impossible timelines. Result: 26 honeypots flagged, 0% in the submitted top 100.

---

## Results

| Metric | Value |
|--------|-------|
| Candidates scored | 100,000 |
| Runtime (2-core Colab CPU) | ~70 seconds |
| Official validator | ✅ Submission is valid |
| Honeypot rate in top 100 | 0% |
| Hard disqualifications in top 100 | 0 |
| Rank 1 score | 0.8858 |
| Rank 10 score | 0.8102 |
| Rank 100 score | 0.7431 |

Top-10 candidates are all genuine Senior ML/NLP/AI Engineers at real Indian product companies (Swiggy, Flipkart, Zomato, Krutrim, Mad Street Den), with all four required signals demonstrated in `career_history` narration and `career_relevance = 1.0` for all.

---

## Compute constraints compliance

| Constraint | Status |
|------------|--------|
| CPU-only (no GPU) | ✅ |
| No network during ranking | ✅ |
| Under 5-minute budget | ✅ (~70s actual) |
| 100 rows, ranks 1–100 | ✅ |
| Scores non-increasing by rank | ✅ |
| Reproducible (byte-identical output across runs) | ✅ verified by diff |

---

## AI tools declaration

Claude (Anthropic) was used in two ways:

1. **Phase 1 (spec extraction):** Collaborative extraction of `jd_spec_fixed.json` from the prose JD, and verification of the company founding-year reference table against the full candidate pool.
2. **Phase 2 (code review):** Diagnosing bugs in the scoring notebook after real test runs against the candidate data, implementing fixes, and validating them empirically before applying. Bugs caught and fixed include: unthresholded honeypot over-flagging (281 → 26), `score_bonus_signals` giving uncorroborated skill mentions full credit, and `recent_langchain_only` disqualifier checking the wrong text source for pre-LLM evidence.

No candidate data was sent to any LLM as part of the scoring pipeline. Claude's involvement was code review and Phase 1 spec authoring — not part of the CPU-only, no-network ranking step.

---

## Pre-computation note

`jd_spec_fixed.json` and `company_founding_years.json` were produced offline (Phase 1) before the scoring notebook runs. They are static files committed to this repo. The scoring step reads them as inputs and does not regenerate them — so `pre_computation_required: true` in the metadata, but the 5-minute budget applies only to Phase 2 (the scoring step), which comfortably fits within it.