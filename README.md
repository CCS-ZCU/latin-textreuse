# Latin Textreuse

Detection of **Bible text reuse in the Latin *Registrum* of Pope Gregory VII**.

This repository contains the complete research pipeline used to identify
quotations and paraphrases of the **Latin Vulgate Bible** within the *Registrum
epistolarum* of **Pope Gregory VII** (GreLa work `cc_10265`), combining
sentence-level semantic embeddings with character-level similarity scoring.

---

## Description

The *Registrum* of Gregory VII is a major eleventh-century collection of papal
letters, and biblical citation was a central instrument of its rhetoric of
authority. This repository automatically detects where the text reuses the
Vulgate — from near-verbatim quotations down to looser paraphrases.

The pipeline works in four stages (see `scripts/`):

1. **Extraction** — Vulgate and register sentences/tokens are pulled from the
   [GreLa](https://github.com/CCS-ZCU/grela) corpus (`grela_v0.7.duckdb`).
2. **Embeddings** — each sentence is encoded with
   [`julian-schelb/multilingual-e5-large-emb-lat-intertext-v1`](https://huggingface.co/julian-schelb/multilingual-e5-large-emb-lat-intertext-v1),
   a 768-dimensional multilingual E5 model fine-tuned for Latin intertextuality.
3. **Match detection** — a FAISS cosine nearest-neighbour index retrieves the
   closest Vulgate sentence for every register sentence.
4. **n-gram scoring** — for finer-grained reuse, sentences are split into
   n-gram windows that are scored with a combination of embedding similarity,
   Jaccard overlap, Levenshtein ratio, and length ratio.

Detected matches are saved as parquet tables and optionally published to
Google Sheets for collaborative inspection.

### Data model

| File | Content |
|------|---------|
| `data/large_files/vulgate_df.parquet` | Sentence-level Vulgate dataframe with token lists. |
| `data/large_files/register_df_with_embeddings.parquet` | Register sentences with embeddings. |
| `data/large_files/vulgate_embeddings.npz` | L2-normalized Vulgate embedding matrix `(N, 768)`. |
| `data/register_matches_vN.parquet` | Per-sentence Vulgate match tables (`sentence_id, subwork_id, text, vulgate_text, score, citation, vulgate_sentence_id`). |
| `data/register_ngram_matches.parquet` | n-gram-level combined-score matches incl. `jaccard_sim`, `levenshtein_distance`. |
| `data/ngrams_dfs/<sentence_id>.json` | Per-sentence n-gram windows with all score lists. |
| `figures/` | Publication-ready plots (per-book match counts, t-SNE of subworks). |

> **Note for agents:** a machine-oriented companion to this document lives in
> [`AGENTS.md`](AGENTS.md) with the full environment setup and code conventions.

## Getting started

The project is developed as a set of Jupyter notebooks. There is no build step
and no test suite — "running the project" means executing the notebooks in
order inside `scripts/`.

### 1. Clone and prepare the environment

```bash
git clone https://github.com/CCS-ZCU/latin-textreuse.git
cd latin-textreuse
python -m venv textreuse_venv
source textreuse_venv/bin/activate
pip install -r requirements.txt
```

### 2. Prerequisites

- **GreLa DuckDB** — the notebooks expect the GreLa corpus at
  `/srv/data/grela/grela_v0.7.duckdb` (a living dataset; newer versions may be
  available). The connection is read-only.
- **GPU (optional but recommended)** — the notebook encoding step defaults to
  `device="cuda"` with `batch_size=256`.
- **Google Sheets publishing (optional)** — a service-account key at
  `~/ServiceAccountsKey.json` is used to publish match tables to a shared
  spreadsheet. The notebooks degrade gracefully if the key is absent.

### 3. Run the pipeline

Open the notebooks in `scripts/` and run them in numbered order:

| Notebook | Stage |
|----------|-------|
| `1_labse-vulgate-register_embeddings.ipynb` | Extract Vulgate + register sentences; compute and save embeddings. |
| `2_textreuse-detection.ipynb` | FAISS retrieval; per-sentence match detection → `data/register_matches_vN.parquet`. |
| `3_textreuse-ngrams.ipynb` | n-gram windows; combined scoring → `data/ngrams_dfs/` + figures. |
| `4_additional-metrics.ipynb` | Extra Jaccard / Levenshtein metrics on match tables. |

The other notebooks are demos and experiments (`grela-demo.ipynb`,
`labse-demo.ipynb`, `llm_translation.ipynb`).

## How to cite

[Once a release is created and published via Zenodo, put its citation here.]

## Acknowledgement

[This work has been supported by ... — e.g. a grant agency if applicable.]

## Related

- [GreLa](https://github.com/CCS-ZCU/grela) — the Latin/Greek corpus database
  used as the underlying data source.

## License

CC-BY-SA 4.0 — see [LICENSE.md](LICENSE.md).
