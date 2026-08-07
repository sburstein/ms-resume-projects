# Shared Conventions

**Every PRD in this directory assumes this document. An agent building any project reads this first,
then the project PRD.** Do not restate these rules in a PRD; deviations are called out explicitly per
project.

---

## 1. Guardrails (non-negotiable)

1. **No Morgan Stanley code, data, output, screenshots, or figures enter any repo here.** Every
   number must trace to a public source recorded in `sources.yaml`.
2. `~/Downloads/el_nino_research_handoff.md` is entitlement-gated MS Research. Never read it into a
   repo, never quote it, never cite it.
3. **Never fabricate data.** If a source is unavailable, rate-limited, or returns nothing, the
   correct behavior is to stop, log it, and report. Do not synthesize plausible values to make a
   pipeline run. Do not "estimate" a number that was supposed to be fetched.
4. **Never retrofit a result.** If a computed figure differs from anything on the résumé, the
   computed figure wins and gets published as-is.
5. Secrets live in `.env`, loaded via `python-dotenv`, and `.env` is in `.gitignore`. API keys are
   never printed, logged, or committed. `.env.example` documents required keys with dummy values.
6. Any request to a government API sets a descriptive `User-Agent` with a contact email. SEC
   specifically requires this and will block otherwise.

---

## 2. Stack

- **Python 3.12.** `uv` for dependency management (`uv init`, `uv add`, `uv run`). Fall back to
  `pip` + `pyproject.toml` if `uv` is unavailable.
- **CLI:** `typer`. Every project exposes `python -m <package> <command>`.
- **Config:** YAML files in `config/`, loaded into `pydantic-settings` models. No magic constants in
  code.
- **Validation:** `pandera` schemas on every dataframe crossing a module boundary.
- **Logging:** `structlog`, JSON to file and human-readable to console. No bare `print`.
- **Data:** `pandas` + `pyarrow`. `geopandas` + `shapely` for vector. `xarray` + `rioxarray` +
  `zarr` for gridded. `duckdb` when a join outgrows memory.
- **Plotting:** `matplotlib` only. One `viz/style.py` module defining the palette and rcParams,
  imported by every figure script. No seaborn defaults, no chartjunk.
- **Testing:** `pytest`. `ruff` for lint and format. `mypy` on `src/` in non-strict mode.
- **CI:** one GitHub Actions workflow running `ruff check`, `mypy`, and `pytest` on push.

---

## 3. Repo layout

```
<repo>/
  README.md               question, data, method, finding, in under 400 words
  FINDINGS.md             the honest result, including what did not work
  METHODOLOGY.md          every assumption, every formula, every choice and why
  CHANGELOG.md
  pyproject.toml
  Makefile
  .env.example
  config/
    sources.yaml          the whitelist, see section 4
    params.yaml           model and run parameters
  src/<package>/
    ingest/               one module per source, each returns a validated dataframe
    transform/
    model/
    viz/
    cli.py
  data/
    raw/                  immutable, gitignored, hash-manifested
    interim/              gitignored
    processed/            gitignored except small outputs
  outputs/
    figures/              committed, regenerable
    tables/               committed if small
    releases/YYYY-MM/     versioned, never overwritten
  tests/
    fixtures/
    test_*.py
  notebooks/              exploration only, never in the critical path
```

---

## 4. `sources.yaml` contract

Every external data source is declared before it is fetched. The fetch layer refuses any URL whose
host is not declared. Schema:

```yaml
sources:
  - id: eia_861
    name: EIA Form 861 Annual Electric Power Industry Report
    url: https://www.eia.gov/electricity/data/eia861/
    host: www.eia.gov
    licence: US Government Work, public domain
    cadence: annual
    lag_months: 11
    accessed: 2026-08-07
    notes: Schema differs between vintages; see ingest/eia861.py
```

`ingest.fetch(source_id, path)` is the only function permitted to make network calls. It checks the
whitelist, caches to `data/raw/<source_id>/`, records a SHA-256 of every downloaded file in
`data/raw/manifest.json`, and short-circuits if the hash is unchanged.

---

## 5. Definition of done

A project is complete when all seven hold. An agent must not declare a project finished otherwise.

1. `make all` regenerates every figure and table in `outputs/` from `data/raw/` with no manual steps.
2. `make test` passes, including at least one **golden-number regression test** that fails if a
   headline figure moves beyond tolerance without a `CHANGELOG.md` entry.
3. `README.md` states the question, data, method, and finding in under 400 words.
4. `FINDINGS.md` records the honest result, the uncertainty, and at least one thing that did not work.
5. `METHODOLOGY.md` documents every assumption and formula, sufficient for someone else to reproduce.
6. At least one figure in `outputs/figures/` is good enough to screen-share in an interview.
7. `sources.yaml` covers every source actually used, and `data/raw/manifest.json` is populated.

---

## 6. Agent execution protocol

- **Work milestone by milestone.** Complete M0 fully, run its acceptance check, commit, then start
  M1. Do not jump ahead.
- **Commit per milestone**, message `M<n>: <milestone title>`. Never commit `data/raw/`.
- **Subset first.** During development, sample every dataset to at most 1,000 rows or one state or
  one month. Run the full dataset only at the milestone acceptance step. This is the single largest
  control on both wall-clock time and token consumption.
- **Stop and report when blocked.** A source that 403s, a schema that changed, a licence that
  forbids scraping: stop, write the finding to `FINDINGS.md`, report to the user. Do not work around
  it by inventing data or by switching to an undeclared source.
- **Never silently drop rows.** Any filter that removes records logs the count and the reason, and
  the counts appear in the run summary.
- **Ask before scraping.** If a source has no API and terms of service are unclear, stop and ask.
- **Model mix.** Sonnet 5 for ingest modules, schema definitions, tests, CLI wiring, docs, and
  plotting. Escalate to Opus for the modelling milestone in each PRD (flagged per project) and after
  two failed debugging attempts on the same error.

---

## 7. Figure standards

- Every figure has a title stating the finding, axis labels with units, a source line, and a
  generation date.
- Colourblind-safe palette. No red/green as the sole distinction.
- Every figure is produced by a script in `src/<package>/viz/`, never by hand, never in a notebook
  that is not run by `make`.
- Save both `.png` at 200 dpi and `.svg`.

---

## 8. Uncertainty reporting

Any headline number is reported with an interval or an explicit statement that no interval was
computed and why. Point estimates without qualification are not acceptable output for any project
here. Where a result rests on an arbitrary methodological choice, run the alternative and report the
spread.
