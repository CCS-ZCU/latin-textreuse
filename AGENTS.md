# Latin Textreuse — Agent Instructions

Always-on guidelines for any AI agent working in this repository (GitHub Copilot, Claude Code, opencode, etc.).

## Project Overview

Research repo for detecting **text reuse of the Latin Vulgate Bible** in the
*Registrum* of **Pope Gregory VII** (GreLa work `cc_10265`).

Pipeline: Vulgate + register sentence extraction from GreLa → `julian-schelb/multilingual-e5-large-emb-lat-intertext-v1` embeddings → FAISS cosine nearest-neighbour search → optionally n-gram windows with combined scoring (embedding / Jaccard / Levenshtein / length-ratio).

The notebooks in `scripts/` are the numbered pipeline stages — **run them in order**. There is no test suite and no build step.

## Environment & Infrastructure

- Python venv: **`textreuse_venv/`** — activate it before running anything: `source textreuse_venv/bin/activate`.
- GreLa DuckDB: `/srv/data/grela/grela_v0.7.duckdb` (opened read-only).
  - ⚠️ GreLa is a **living, team-maintained dataset**. Verify the current version with `ls -lh /srv/data/grela/` and don't hardcode v0.7 assumptions (stats, ids) without checking.
- Embedding model: `julian-schelb/multilingual-e5-large-emb-lat-intertext-v1` (768-dim, recommended L2-normalized, `device="cuda"`, `batch_size=256` for large encodes).
- Google Sheets publishing (for match tables): `~/ServiceAccountsKey.json` + gspread; spreadsheet `https://docs.google.com/spreadsheets/d/1QroTEQ9gQf9cLO9mvolp7fELNbYYjgvGd48yAiTj03w`.
- `data/large_files/` and `textreuse_venv/` are **git-ignored**.

## Repository Layout

| Path | Role |
|------|------|
| `scripts/1_labse-vulgate-register_embeddings.ipynb` | Extract Vulgate + register sentences/tokens from GreLa; build `vulgate_df` / `register_df`; compute and save embeddings. |
| `scripts/2_textreuse-detection.ipynb` | FAISS retrieval; per-sentence Vulgate match detection; writes `data/register_matches_v*.parquet` + publishes sheet. |
| `scripts/3_textreuse-ngrams.ipynb` | n-gram windows per register sentence → per-sentence best-verse matches (`detect_multiple_vulgate_matches_with_ngrams`) → `matches_long_df`; combined scoring; writes per-sentence JSON to `data/ngrams_dfs/`, combined results, and figures. |
| `scripts/4_additional-metrics.ipynb` | Extra metrics (Jaccard, Levenshtein) on match tables. |
| `scripts/grela-demo.ipynb`, `labse-demo.ipynb` | Small GreLa / model demos. |
| `scripts/llm_translation.ipynb` | LLM translation experiments. |
| `data/` | Inputs + tracked result tables (parquet). |
| `data/large_files/` | Big intermediates (git-ignored). |
| `data/ngrams_dfs/` | Per-sentence n-gram JSONs. |
| `figures/` | Saved plots. |
| `README.md` | Human-oriented overview (setup, pipeline, data model) — see AGENTS.md for agent-focused details. |

## Pipeline Notes (recovered 2026-09-25 — do not delete)

Notebook 3's ngram stage has a **three-level chain**; the middle step is easy to
delete by accident (it was cleared from cell source once and restored from git 615cc22):

1. **ngram level** — `build_ngrams_df` + `score_ngram_matches` build per-ngram
   windows and save raw scores to `data/ngrams_dfs/<sentence_id>.json`.
2. **sentence level (Stage 2)** — `detect_multiple_vulgate_matches_with_ngrams`
   (cell `#VSC-b12f2472`) + `register_df["matches"] = ...apply(...)` (cell
   `#VSC-4e312980`). Slices each clean sentence into overlapping
   `window_size=10`/`step=1` windows (+ full sentence), FAISS-searches each
   (`top_k_per_ngram=3`, `min_sim=0.75`), keeps the **best-scoring window per
   distinct Vulgate verse**, sorts by `best_raw_score` desc, caps at
   `max_total=5`.
3. **long format** — `matches_long_df` flattening (one row per matched verse,
   `match_rank_within_sentence`), then `jaccard_sim` / `levenshtein_sim` /
   `combined_score` cell (`#VSC-83f5b334`) and `to_parquet`.

If a cell references `row["matches"]` and that column is missing at runtime,
check that the Stage-2 cell (step 2) survived and ran.

## Data Model (key files & schemas)

- `vulgate_df.parquet` / `register_df*.parquet` — sentence-level dataframes. Each row carries `tokens` (list of token dicts: `token_id, token_text, lemma, pos, char_start, char_end` + `ref`-derived fields), `sent_text_clean` (normalized), and (register) `register_ref` / `embedding`.
- `vulgate_embeddings.npz` — key `embeddings` (float32, L2-normalized, shape `(N, 768)`).
- `register_matches_v2.parquet` — `sentence_id, subwork_id, text, vulgate_text, score, citation, vulgate_sentence_id`.
- `register_ngram_matches.parquet` — `sentence_id, subwork_id, sentence, score, vulgate_sentence_id, vulgate_title, vulgate_text, ngram, jaccard_sim, levenshtein_distance`.
- `ngrams_dfs/<sentence_id>.json` — rows are n-gram windows; columns `ngram_tokens`, `ngram_lemmata`, `ngram_start`, `ngram_stop`, `length`, `indices`, `embedding_scores`, `jaccard_scores`, `levenshtein_scores`, `vulgate_lens`, `lenratio_scores`.

## Conventions & Gotchas

- **Text normalization** (small helper reused everywhere — keep identical):
  NFD-normalize → drop chars outside `[a-zA-Z\s]` → lowercase → replace `v→u`, `j→i`.
- `cc_*` works link to `https://mlat.uzh.ch/<id>` via the `add_cc_link` helper.
- Vulgate `ref` is a JSON string with `chapter` / `verse`; register `ref` has `parent_pid` from which `register_ref` is derived.
- Register work id: `cc_10265`; Vulgate works: `grela_id LIKE 'vulgate_%'`; optional URL id: `cc_20265`.
- Semantic lemmata filter for reuse analyses: POS in `NOUN, ADJ, VERB, PROPN`.
- **Scoring metrics** (keep definitions identical across notebooks):
  - `best_raw_score` — embedding cosine similarity (FAISS inner product on L2-normalized vectors).
  - `jaccard_sim` — Jaccard over whitespace-token sets.
  - `levenshtein_sim` — **similarity in [0,1]**; in the n-gram pipeline it is `Levenshtein.ratio` and feeds `combined_score`.
  - `levenshtein_distance` — **raw edit distance** (NOT a similarity; never substitute it for `levenshtein_sim`).
  - `combined_score = (best_raw_score + jaccard_sim + levenshtein_sim) / 3` (verified 1:1 against `register_ngram_matches_v4.parquet`).
- The workspace uses shared notebook conventions (see `~/notebooks/` conventions): numbered `N_pipeline.ipynb`, `<domain>_helpers.py` helpers, type hints + dataclasses, DuckDB connection helper with `memory_limit='96GB'` / `temp_directory='/srv/data/duckdb_tmp'`, batch processing for large data.
- GreLa interface tables: `works` (metadata incl. `author, title, not_before, not_after, token_count`), `sentences` (`sentence_id, grela_id, position, sent_text`), `tokens` (`token_id, token_text, lemma, pos, ref, char_start, char_end`).

## Workflow Rules for Agents

1. **Activate the venv** before running Python (`source textreuse_venv/bin/activate`).
2. When tasks involve GreLa data, always verify the DB version/path first.
3. Keep big intermediates in `data/large_files/`; write tracked results to `data/` and plots to `figures/`.
4. Preserve the normalization + scoring functions (Jaccard, Levenshtein via `Levenshtein.ratio`, embedding cosine) — changing them silently changes research results.
5. Add new pipeline stages as numbered notebooks under `scripts/` following the existing conventions, rather than scattering ad-hoc scripts.
