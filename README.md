# Japan M&A Deal Sourcing & Selection Engine
_Optimizes M&A shortlists for the Japan market by turning disclosure feeds into ranked and constraint‑aware selections._

[![CI](https://img.shields.io/badge/CI-GitHub_Actions-lightgrey)](#)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](#license)
[![Python](https://img.shields.io/badge/Python-3.10%2B-blue)](#)
[![Status](https://img.shields.io/badge/status-Active-darkgreen)](#)

> **tl;dr**: Ingest Japan disclosure data (e.g., TDnet/EDINET/J‑Quants), resolve entities, retrieve candidates, re‑rank with ML, and select a **portfolio** of deals via **ILP** under budget/bandwidth/diversity constraints. Export shortlists to CSV/CRM with metrics and traces.

---

## Table of Contents
- [Why this exists](#why-this-exists)
- [Key capabilities](#key-capabilities)
- [Architecture](#architecture)
- [Data sources & compliance](#data-sources--compliance)
- [Quickstart](#quickstart)
- [Configuration](#configuration)
- [Command‑line (CLI)](#command-line-cli)
- [Outputs](#outputs)
- [Evaluation & monitoring](#evaluation--monitoring)
- [Project layout](#project-layout)
- [Performance tips](#performance-tips)
- [Security & privacy](#security--privacy)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)
- [附記 (Japanese note)](#附記-japanese-note)
- [Appendix A — Why this README covers the “best information” (step‑by‑step)](#appendix-a--why-this-readme-covers-the-best-information-step-by-step)
- [Appendix B — Optional extras to add later](#appendix-b--optional-extras-to-add-later)

---

## Why this exists
- **Deal teams** need **faster, higher‑quality shortlists** aligned to buyer theses.
- Japan’s disclosures (TDnet, EDINET, market/alt data) are rich but **fragmented**.
- This project offers a **repeatable pipeline** from raw feeds to **ranked selections** that respect **real‑world constraints** (team capacity, sector mix, fee goals).

---

## Key capabilities
- **Ingestion**: Connectors for Japan corporate/market disclosures (see compliance below).
- **Entity resolution (JP‑specific)**: Normalize names; map **Houjin Bangou** ↔︎ tickers/issuers; deduplicate subsidiaries.
- **Candidate retrieval**: ANN (e.g., FAISS) over buyer profiles & issuer vectors; configurable filters.
- **Re‑ranking (ML)**: Gradient‑boosted trees or similar; probability calibration; thresholding.
- **Portfolio selection (ILP)**: Optimize for expected revenue/score with constraints (budget, headcount, sector limits, buyer coverage).
- **Observability**: Metrics (`metrics.json`), predictions (`predictions.csv`), PR/AUC plots; optional experiment tracking.
- **Exports**: CSV, Parquet; optional CRM/export hooks.

---

## Architecture
```
            +---------------------+
            |  Data Sources       |   TDnet / EDINET / J‑Quants / CRM snapshots
            +----------+----------+
                       |
                       v
            +----------+----------+       Entity keys (Houjin, ticker, LEI)
            | Ingest & Normalize |  --->  Text/num features, embeddings, labels
            +----------+----------+
                       |
                       v
            +----------+----------+       ANN index (FAISS) + filters
            |  Retrieval (ANN)    |  ---> Candidate set per buyer
            +----------+----------+
                       |
                       v
            +----------+----------+       XGBoost/LightGBM + calibration
            |  Re‑ranking (ML)    |  ---> Scored candidates
            +----------+----------+
                       |
                       v
            +----------+----------+       OR‑Tools/PuLP; budget/diversity/capacity
            |  ILP Selection      |  ---> Portfolio shortlist
            +----------+----------+
                       |
                       v
            +----------+----------+       CSV/Parquet/CRM; metrics & plots
            |   Outputs & KPIs    |
            +---------------------+
```

---

## Data sources & compliance
This repo ships **connectors and schemas**, not third‑party data.

- **TDnet / EDINET / J‑Quants**: Use **official APIs/feeds where available** and follow their **terms, rate limits, and attribution**.  
- **Identifiers**: Houjin Bangou (法人番号), JP tickers, ISIN, optional LEI.  
- **Licensing**: Verify redistribution/processing rights before committing datasets.  
- **PII**: Avoid storing personal data. Redact if present in CRM notes.

> ⚖️ Always check provider ToS. The maintainers are not responsible for your data use.

---

## Quickstart

### 1) Prerequisites
- Python **3.10+** (3.11 recommended)
- (Optional) CUDA‑capable GPU for faster training/scoring
- (Optional) Docker for reproducible runs

### 2) Install
```bash
# clone
git clone https://github.com/<you>/deal-sourcing-jp.git
cd deal-sourcing-jp

# create env & install
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -U pip wheel
pip install -e ".[all]"   # uses pyproject/extra deps

# pre-commit (optional)
pre-commit install
```

### 3) Minimal `.env`
Create `.env` at repo root:
```dotenv
# Example: set credentials/tokens as needed by your connectors
TDNET_API_KEY=...
EDINET_API_KEY=...
JQUANTS_API_KEY=...
CRM_DSN=postgresql+psycopg2://user:pass@host:5432/dbname
```

### 4) Minimal config
Save as `configs/config.yaml`:
```yaml
project: dealmatch-jp
random_seed: 42

data:
  cache_dir: data/cache
  raw_dir: data/raw
  processed_dir: data/processed
  buyers_path: data/buyers/buyers.csv        # buyer theses/profiles
  disclosures_path: data/disclosures/        # normalized issuer data
  id_columns: [houjin_bangou, ticker, isin]

features:
  text_fields: [disclosure_title, summary, notes]
  numeric_fields: [revenue, ebitda_margin, leverage, growth]
  embedding_model: "sentence-transformers/all-MiniLM-L6-v2"

retrieval:
  index_path: models/faiss/index.bin
  top_k: 200
  filters:
    sectors_exclude: []
    market_cap_min: 0

ranker:
  algo: xgboost
  params:
    max_depth: 6
    n_estimators: 600
    learning_rate: 0.05
  calibrate: isotonic

selection:
  objective: expected_fee              # or "score"
  constraints:
    budget_hours: 80
    max_per_sector: 10
    min_small_cap: 3
    analyst_capacity: 20
  solver: pulp                         # or "ortools"
  output_size: 30

outputs:
  dir: outputs/
  predictions_csv: predictions.csv
  metrics_json: metrics.json
  export_crm: false
```

### 5) Run end‑to‑end
```bash
# Ingest & normalize (write processed parquet/csv)
python -m dealmatch_jp ingest --config configs/config.yaml

# Build or update retrieval index
python -m dealmatch_jp build-index --config configs/config.yaml

# Train ranker (or skip to predict with a pretrained model)
python -m dealmatch_jp train --config configs/config.yaml

# Score candidates for each buyer
python -m dealmatch_jp score --config configs/config.yaml

# Optimize selection under constraints
python -m dealmatch_jp select --config configs/config.yaml

# Evaluate and produce artifacts in outputs/
python -m dealmatch_jp evaluate --config configs/config.yaml
```

---

## Configuration
All behavior is controlled by a single YAML (see example). Key knobs:

- **features.embedding_model**: switch sentence model without code changes.
- **retrieval.top_k**: breadth of candidate pool before re‑ranking.
- **ranker.params**: XGBoost/LightGBM params; set `calibrate` to `isotonic` or `platt`.
- **selection.constraints**: business rules (budget, capacity, sector mix).
- **objective**: expected fee vs. pure relevance score.

> Tip: keep **environment‑specific** values in `.env` and reference them in the YAML with `${ENV_VAR}` if you use `python‑dotenv` or `pydantic‑settings`.

---

## Command‑line (CLI)
```
python -m dealmatch_jp <command> [--config CONFIG] [options]

Commands:
  ingest         Pulls/reads source data, normalizes schema, writes processed files.
  build-index    Builds ANN index over embeddings/features.
  train          Trains the re‑ranker, saves model and calibration.
  score          Produces per‑buyer candidate scores with calibrated probabilities.
  select         Solves ILP to produce a shortlist portfolio under constraints.
  evaluate       Computes metrics (AUC, AP, Brier), plots, and KPI summaries.
  export         Writes predictions/shortlist to CSV/Parquet/CRM.

Add `-h` to any command for flags.
```

---

## Outputs
- `outputs/predictions.csv` — scored candidates per buyer (ids, scores, probs, features)
- `outputs/shortlist.csv` — ILP‑selected portfolio with rationale & constraint traces
- `outputs/metrics.json` — AUC/AP/Brier, lift@k, coverage, constraint utilization
- `outputs/plots/*.png` — PR/AUC curves, calibration plots, feature importance
- `outputs/run.jsonl` — optional run manifest (hyperparams, seeds, versions)

---

## Evaluation & monitoring
- **Ranking**: AUC/PR‑AUC, **lift@k**, recall at buyer quotas.
- **Calibration**: reliability curves; Brier score; expected vs. realized pickup.
- **Selection**: objective value vs. budget; constraint slack; sector/buyer coverage.
- **Business KPIs**: proxy revenue (discounted expected fees), pipeline velocity.

> For recurring runs, integrate MLflow/W&B or write metrics to a warehouse for dashboards.

---

## Project layout
```
.
├── configs/                  # YAML configs
├── data/                     # raw/processed caches (gitignored)
├── notebooks/                # exploratory & reports
├── outputs/                  # predictions, metrics, plots (gitignored)
├── src/dealmatch_jp/         # package code
│   ├── __init__.py
│   ├── ingest/               # connectors, parsers, normalizers
│   ├── features/             # featurization, embeddings
│   ├── retrieval/            # FAISS index, filters
│   ├── ranker/               # training, calibration, scoring
│   ├── selection/            # ILP model & solvers
│   ├── export/               # csv/crm writers
│   └── cli.py                # CLI entrypoints
├── tests/                    # unit/integration tests
├── pyproject.toml            # deps & build
├── Makefile                  # common tasks
└── README.md
```

---

## Performance tips
- **GPU**: Enable GPU for training/scoring to cut runtimes significantly.
- **Index**: Tune **`nlist`/`nprobe`** (IVF) or switch to HNSW for retrieval speed/recall tradeoffs.
- **Batching**: Vectorize featurization & scoring; avoid Python loops.
- **Caching**: Cache expensive embeddings and prefiltered candidate pools.

---

## Security & privacy
- Avoid committing keys or data. Use `.env` and secret managers.
- If using CRM notes, strip PII and apply least‑privilege DB access.
- Log **aggregates**, not raw texts, when possible.

---

## Roadmap
- [ ] LightGBM / CatBoost rankers
- [ ] Cross‑buyer fairness/coverage constraints in ILP
- [ ] Automated thresholding via cost curves
- [ ] Richer CRM exporter (bi‑directional sync)
- [ ] Streaming incremental updates

---

## Contributing
Issues and PRs welcome! Please run:
```bash
make fmt && make lint && make test
```
and include a brief description, reproduction steps, and screenshots of new plots if applicable.

---

## License
MIT — see `LICENSE` for details.

---

## 附記 (Japanese note)
本リポジトリは、日本の開示データを活用した**M&Aディールの候補抽出・再ランキング・最終選定**を自動化するためのリファレンス実装です。各データ提供者の**利用規約**を順守してください。

---

## Appendix A — Why this README covers the “best information” (step‑by‑step)

1) **One‑line value prop** — The opening line states what it does and for whom (Japan M&A).  
2) **Key capabilities** — Highlights the unique aspects (Houjin Bangou mapping, retrieval → re‑rank → ILP).  
3) **Architecture sketch** — Visual pipeline helps new contributors orient quickly.  
4) **Data & compliance** — Prevents ToS violations and sets expectations on data rights/PII.  
5) **Quickstart** — Frictionless end‑to‑end run with `.env`, YAML, and CLI.  
6) **Config‑driven design** — Shows how to adapt behavior without editing code.  
7) **CLI reference** — Encapsulates pipeline steps for automation and CI.  
8) **Outputs & metrics** — Names concrete files and what’s inside for downstream wiring.  
9) **Evaluation** — Ranking, calibration, and selection KPIs for auditability.  
10) **Project layout** — Onboarding clarity and contribution norms.  
11) **Performance & security** — Practical tips to run fast and stay compliant.  
12) **Roadmap & contributing** — Signals maintenance and sets PR quality expectations.

---

## Appendix B — Optional extras to add later
- **`model_card.md`** — assumptions, risks, data lineage, limitations  
- **`docs/` site** (mkdocs/sphinx) — API docs and examples  
- **`docker-compose.yml`** — local Postgres + notebook for reproducible demos  
- **`examples/`** — toy buyers/disclosures for a tiny runnable sample
