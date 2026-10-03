# Methodex — Economic-Methodology MCP

**The construction-and-revision history of every economic statistic.** Methodex answers a question most data portals ignore: *what methodology governed a statistic at a given point in time, and how did it change?* It is both a **public website** and an **MCP server** ("Context7 for economic methodology") serving the documented lineage of how official economic statistics are constructed and revised.

Repo: `andenick/methodex-web` (verified 2026-10-02).

---

## Public-domain-only posture

This repository and both services serve **only the public-domain US-federal-government layer** of the Methodex corpus:

- **957 documents / 392 revision events / 80 statistics / 14 measures.**
- Enforced two ways: (1) `METHODEX_PUBLIC_ONLY=1` is forced *at import time* (`webapp/app/config.py`, `src/methodex_mcp.py`) so the in-memory stores are physically filtered to `license == "US-Gov public domain"` before any tool or page can read them; and (2) the corpus state shipped in the repo (`Technical/campaign/state/*.json`) is already pre-filtered to the public-domain subset — no in-copyright, international, or academic material is present on disk.

## Architecture

Two services share one query layer:

| Service | Port | What it is |
|---|---|---|
| **methodex-web** | 8080 | FastAPI + Jinja2 + Plotly public website (`webapp/`) |
| **methodex-mcp** | 8000 | The 13 methodology tools over the **Streamable-HTTP** MCP transport at path `/mcp` (`src/methodex_mcp_http.py`) |

The canonical tool logic lives once in `src/methodex_mcp.py`; the website services and the MCP HTTP entrypoint both reuse it, so web and MCP answer identically. The website reads pre-built public-domain caches (`webapp/site_data/cache/*.parquet`) and offers the packaged data at `webapp/site_data/downloads/methodex_public_data.zip`. Shared chrome is vendored from the Arcanum Site Kit (see `VENDORED_FROM.txt`) so the site builds standalone; no CDN dependency.

## The 13 MCP tools

`resolve_statistic(query)` · `get_methodology(statistic_id, as_of_date)` — **the killer tool**: what methodology governed the statistic *then* · `get_revision_history(statistic_id, from_year, to_year, verified_only)` · `diff_methodology(statistic_id, date_a, date_b, verified_only)` · `search_methodology(query)` · `get_document(md5)` · `get_concept_history(concept)` — every event touching a concept (e.g. `owners_equivalent_rent`, `hedonic`) · `get_table_history(table_id)` — how a published output table was defined over time (e.g. `BEA.NIPA.T1.1.5` = GDP) · `list_measures(statistic_id, section)` · `get_vintage_data(statistic_id, as_of_date)` — cross-links methodology to the underlying data vintage (ALFRED/FRED) · `get_methodology_timeline(statistic_id, fmt)` · `methodex_status(statistic_id)` · `semantic_search(query)` — TF-IDF cosine ranking, no external API.

Every response carries provenance: governing-document md5 + page citation.

## Run it

```bash
docker compose up --build
# website:  http://localhost:8080
# MCP:      http://localhost:8000/mcp  (streamable-http)
```

Locally (Python 3.13):

```bash
pip install -r webapp/requirements.txt
PYTHONPATH="webapp:src" METHODEX_PUBLIC_ONLY=1 \
  gunicorn app.main:app --chdir webapp -k uvicorn.workers.UvicornWorker -b 0.0.0.0:8080
PYTHONPATH="src" METHODEX_PUBLIC_ONLY=1 python src/methodex_mcp_http.py   # MCP server
METHODEX_PUBLIC_ONLY=1 python src/methodex_mcp.py demo                    # CLI demo, no server
```

## Anu replication package

The [`anu/`](anu/) directory contains a complete data-replication package: `series_registry.json` (the canonical data contract), fetch/process/validate scripts, and Data Provenance Records. See [`anu/README.md`](anu/README.md) to reproduce the data.

## License

MIT (provisional — final license pending an open decision). See `LICENSE`.
