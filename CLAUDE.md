# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A VinUni AICB Day 19 lab (Track 2): a hybrid search API (BM25 + vector + RRF)
backed by Qdrant, plus a Feast feature store, taught through 8 Jupytext
notebooks (`notebooks/*.py`) that exercise the `app/` modules. There are two
parallel infra paths — **lite** (in-memory Qdrant + SQLite Feast, no Docker)
and **docker** (Qdrant/Redis/Postgres via `docker-compose.yml`) — selected via
`.env` (`QDRANT_MODE=memory|server`) and sharing the same `app/` code and
`qdrant-client` API.

## Commands

```bash
bash setup-lite.sh      # venv + deps + seed corpus + smoke test (~60s, first time)
bash setup-docker.sh    # full stack: Docker services + venv + seed (~3-8 min)

make test               # pytest -q (app/ + scripts/, ~2s)
make api                # uvicorn app.main:app --reload --port 8000
make benchmark           # Precision@10 (kw/sem/hybrid) + P99 latency table
make seed                # regenerate data/corpus_vn.jsonl + data/golden_set.jsonl
make gen-advanced        # regenerate data for NB6 (agent queries) + NB8 (spend parquet)
make lab                 # convert notebooks/*.py -> .ipynb, launch Jupyter Lab :8888
make notebooks           # execute ALL notebooks headless in place (what the grader runs)
make verify-lite         # 5s smoke test of the lite stack
make runtime-check       # report which of docker/podman/apple-container is available
```

Single test file/function: `.venv/bin/pytest tests/test_search.py -q` or
`-k <name>`. Tests live in both `tests/` and are also collected from `app/`
and `scripts/` per `[tool.pytest.ini_options] testpaths` in
[pyproject.toml](pyproject.toml).

`make clean-lite` wipes venv, generated data, and the Feast registry — use
when Feast state (`registry.db`) gets corrupted or `feast apply` errors
inexplicably.

Notebooks are **Jupytext `.py` files** (the source of truth, under
`notebooks/`) auto-converted to `.ipynb` by `make lab` / `make notebooks`. Edit
either representation — Jupytext keeps them in sync — but don't hand-edit a
generated `.ipynb` if the paired `.py` also changed, and vice versa without
reconverting.

## Architecture

**Core retrieval path** ([app/search.py](app/search.py)): `Searcher` builds a
BM25 index (`rank_bm25`) and a Qdrant collection over the same 1000-doc
corpus at construction time, then serves `keyword` / `semantic` / `hybrid`
modes. Hybrid fuses the two via **Reciprocal Rank Fusion** (`k=60`,
1-based ranks): pull `top_k*5` (min 50) from each retriever, sum
`1/(k+rank)` per doc, re-sort. [app/main.py](app/main.py) wraps this in a
single FastAPI `GET /search` endpoint; the `Searcher` is built once at
startup (lifespan) since embedding + indexing 1000 docs is the expensive
part — reuse it, don't rebuild it per-request.

**Embeddings are pluggable** ([app/embeddings.py](app/embeddings.py)):
`Embedder` reads `EMBEDDING_BACKEND` (`fastembed` default / `multilingual` /
`bge-m3` / `openai`) and exposes a uniform `.embed(texts) -> Iterator[ndarray]`
regardless of provider (fastembed / sentence-transformers / openai SDK
underneath). **Dimension follows the model** (384/1024/1536) — never assume
384. Switching backends means re-indexing; nothing auto-migrates a
Qdrant collection across dimensions.

**Advanced modules (NB5–NB8)** build on the same `Searcher`/corpus rather than
re-embedding:
- [app/filters.py](app/filters.py) — `FilteredIndex.from_searcher()` clones
  vectors out of the base Qdrant collection (`with_vectors=True`) into a
  richer, filter-indexed collection. Demonstrates three filtering strategies
  (post-filter recall cliff vs. pre-filter exact-but-unindexed vs.
  filtered-ANN) side by side.
- [app/metadata.py](app/metadata.py) — deterministic per-doc metadata
  (tenant/access/published date) derived by hashing `doc_id`, *not* drawn from
  the corpus RNG stream, so adding fields never perturbs the existing
  golden-set-dependent corpus.
- [app/agent.py](app/agent.py) — a rule-based (no-LLM, no API key) planner
  that decomposes multi-intent questions, calls a `search_docs` tool, and
  retries once with filters relaxed if evidence is thin. `build_context()` is
  where retrieval (Qdrant) meets personalization (Feast online features).
- [app/cache.py](app/cache.py) — semantic cache over a Qdrant collection with
  three independently tunable/breakable knobs: similarity threshold (false
  hits), TTL (staleness), and tenant namespacing (`namespaced=False` exists
  *only* to let students observe a cross-tenant leak before fixing it — never
  set that outside the exercise).
- [app/features.py](app/features.py) — six feature families over a synthetic
  event log, plus two intentional leakage demos (naive vs. in-fold target
  encoding; `latest_join` vs. `pit_join` point-in-time joins) scored with a
  pure-numpy `auc()`.

**Feast** ([app/feast_repo/](app/feast_repo/)): three feature views
(`user_profile_features`, `item_popularity_features`,
`query_velocity_features`) reading Parquet `FileSource`s built by NB4.
[app/feast_repo_ondemand/](app/feast_repo_ondemand/) holds a separate repo
for the on-demand feature view (`amount_vs_avg`) — note the import gotcha
documented at the top of
[definitions.py](app/feast_repo_ondemand/definitions.py): `on_demand_feature_view`
must be imported from `feast.on_demand_feature_view`, not from `feast`
directly. `feature_store.yaml` in each repo switches between SQLite
(lite) and Redis+Postgres (docker) — see the commented-out block in
[feature_store.yaml](app/feast_repo/feature_store.yaml).

**Data flow**: nothing under `data/` is committed — `scripts/seed_corpus.py`
deterministically generates `corpus_vn.jsonl` (1000 VN docs) and
`golden_set.jsonl` (50 queries) from a seeded RNG; `scripts/gen_agent_queries.py`
and `scripts/gen_spend.py` generate the NB6/NB8 inputs. Regenerate with
`make seed` / `make gen-advanced`, not by hand.

## Non-obvious constraints

- **Python 3.14**: `overrides-py314.txt` (pinning `dill` away from a version
  that crashes serializing Feast's on-demand-feature-view UDF) is applied
  automatically by `setup-lite.sh` only when the venv is actually 3.14 —
  don't apply it unconditionally.
- **RRF ranks are 1-based**, not 0-based — a common bug source when
  re-implementing fusion logic.
- Qdrant's **local in-memory client silently ignores payload indexes** and
  warns on every call in [app/filters.py](app/filters.py) — that warning is
  deliberately suppressed there; it matters (real speed gain) only against a
  Qdrant server.
- Windows dev note: `Makefile` targets assume a POSIX shell + `.venv/bin/*`
  paths; on Windows use Git Bash (the Bash tool) or adapt paths to
  `.venv/Scripts/*` when invoking tools directly.
