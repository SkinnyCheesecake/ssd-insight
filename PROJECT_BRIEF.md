# ssd-insight — Project Brief & 7-Day Execution Plan

**Project name:** `ssd-insight` — Data-Center SSD Fleet Telemetry & Health Analyzer
**Owner:** <your name>
**Repo:** `ssd-insight/`
**Interview date:** Wednesday 2026-10-21
**Work window:** Wed 2026-10-14 → Tue 2026-10-20 (7 days)

> This file is your single source of truth. If you get lost or behind, re-read
> **Section 3 (Scope)** and **Section 10 (Cut Lines)** before writing any code.

---

## 1. The pitch (memorize this)

> "ssd-insight is a Python service and CLI that ingests telemetry and SMART data
> from a fleet of data-center SSDs, stores it, scores each drive's health,
> forecasts when it will run out of endurance, flags abnormal drives with
> statistics plus a light ML model, and exposes it all through a REST API."

**One project, one sentence, and it touches every single line of the job
description.** That is the whole point. Do not add a second project.

---

## 2. Why this project wins the interview (feature → JD mapping)

| Solidigm JD bullet | What proves it in your project |
| --- | --- |
| Design/develop/test/maintain software tools & back-end systems | FastAPI service + CLI + pytest test suite |
| Apply **Python** to build scripts/services/automation | Entire project is Python; CLI automates ingest + reporting |
| Support **AI and data analysis**: analyze data/logs/metrics, find trends | Telemetry analytics engine + anomaly detection |
| Troubleshoot/debug; reproduce problems; **root-cause analysis** | Anomaly report ties a symptom (health drop) to a root cause (e.g. media errors climbing) |
| Work in **Linux** to build/test/deploy | Makefile, Dockerfile, shell usage, systemd-friendly CLI |
| Work with **cross-functional** teams | Write docs aimed at firmware/data-science/product readers |
| Create clear **documentation** | README, architecture doc, auto-generated Markdown reports |
| Back-end development, **APIs**, service-oriented architecture | REST API with versioned routes + OpenAPI docs |
| Exposure to **AI/ML** concepts | Robust z-score **and** Isolation Forest (scikit-learn) |
| **Data analysis**, scripting, automation | Parsers, aggregation, endurance forecasting |
| **Git** | Daily conventional commits, feature branches, PRs |
| Interest in **data-center storage** | The domain *is* NVMe/SMART/endurance/thermals |

**The differentiator:** most candidates show a generic CRUD app. You show a
*storage-domain* tool that a firmware or SSD-validation engineer would actually
use. Say that out loud in the interview.

---

## 3. Scope — build the MVP first, stretch later

### MVP (must work to demo — target end of Day 4)
1. **Synthetic telemetry generator** (you need data; you have no real SSD logs).
2. **Parsers**: CSV + JSON Lines (and one smartctl-style text log).
3. **SQLite storage layer** with indices.
4. **Analytics**: health score + endurance projection + temperature/error stats.
5. **CLI**: `generate`, `ingest`, `list`, `summary`, `health`, `anomalies`, `report`.
6. **REST API** (FastAPI) with OpenAPI docs.
7. **Tests** (pytest) + **README**.

### Stretch (only if MVP is green and polished)
8. Isolation Forest anomaly detection (scikit-learn).
9. Markdown fleet report + per-drive report.
10. Dockerfile + GitHub Actions CI.
11. Tiny web dashboard for health (keep it minimal).
12. Prometheus metrics endpoint.

> Rule: a **small, complete, tested** project beats a big broken one. Every time.

---

## 4. Architecture (target layout)

```
ssd-insight/
├── PROJECT_BRIEF.md          # this file
├── README.md                 # pitch, install, quickstart, screenshots/asciinema
├── pyproject.toml            # package metadata + deps
├── requirements.txt          # runtime deps
├── requirements-dev.txt      # pytest, httpx, ruff, scikit-learn
├── Makefile                  # install / test / lint / run / report
├── Dockerfile                # Linux deploy story
├── .gitignore                # ignore .venv, *.db, __pycache__, sample output
├── .github/workflows/ci.yml  # run pytest on push (stretch)
├── docs/
│   ├── architecture.md
│   └── interview-notes.md    # your talking points + Q&A rehearsal
├── src/ssd_insight/
│   ├── __init__.py
│   ├── config.py             # thresholds (temp, spare, wear), DB path
│   ├── models.py             # Sample, DriveInfo, HealthReport dataclasses
│   ├── db.py                 # SQLite: schema, insert, query
│   ├── parsers.py            # CSV / JSONL / smartctl text
│   ├── synthetic.py          # fleet simulator -> sample CSV
│   ├── analytics.py          # health score, wear forecast, stats
│   ├── anomalies.py          # robust z-score + Isolation Forest
│   ├── report.py             # Markdown report rendering
│   ├── cli.py                # argparse subcommands
│   └── api/
│       ├── __init__.py
│       └── main.py           # FastAPI app + routes + OpenAPI
├── tests/
│   ├── test_parsers.py
│   ├── test_db.py
│   ├── test_analytics.py
│   ├── test_anomalies.py
│   └── test_api.py
└── sample_data/
    └── telemetry_sample.csv  # generated artifact (can be committed, small)
```

---

## 5. Data model (the fields you track)

One telemetry **sample** per drive per timestamp:

| Field | Meaning |
| --- | --- |
| `timestamp` | UTC ISO-8601 |
| `drive_id` | serial number (primary key of a drive) |
| `model`, `firmware`, `capacity_gb` | inventory |
| `temperature_c` | composite temperature |
| `percent_used` | endurance consumed (0–100), a.k.a. "wear" |
| `available_spare_pct` | NVMe available spare (100 new, drops with wear) |
| `power_on_hours` | cumulative |
| `media_writes_tb` | TBW written (endurance fuel gauge) |
| `host_reads_gb`, `host_writes_gb` | workload |
| `unsafe_shutdowns` | count |
| `media_errors`, `crc_errors`, `reallocated_sectors` | integrity counters |
| `predicted_failure` | 0/1 SMART flag |

**Health score (0–100)** = 100 − sum of explainable penalties:
- wear (`percent_used`), spare headroom, media-error *rate*, thermal exposure
  (fraction of samples above warn temp), unsafe shutdowns, reallocated sectors.
- Grade A–F, status `ok` / `warning` / `critical`.
- **Always print the penalty breakdown** — explainability is the selling point.

**Endurance forecast:** least-squares fit of `percent_used` vs time → wear rate
(%/day) → `days_to_threshold`. Report a confidence hint from the fit quality.

---

## 6. Working rules (how you must work)

1. **One-time setup first (Day 1, before any feature code):**
   ```bash
   cd ~/ssd-insight
   git init
   python3 -m venv .venv && source .venv/bin/activate
   python -m pip install --upgrade pip
   # deps go in requirements*.txt; install as you need them
   ```
2. **Git discipline.** Commit every work session with a conventional message:
   `feat: add csv telemetry parser`, `fix: handle missing smart fields`,
   `test: cover health score thresholds`, `docs: add architecture diagram`.
   Use a branch per feature (`feat/anomaly-detection`) and merge to `main`.
   Your commit history is itself evidence of Git skill — keep it clean.
3. **Write the test at the same time as the code** for `analytics`, `anomalies`,
   `parsers`, and the API. Aim for >70% coverage on core modules. Run
   `pytest -q` before every commit.
4. **Type hints on every public function**, short docstrings, functions < 50 lines.
5. **Run everything on Linux from the CLI first**, then wire it into the API.
   The API is a thin layer over modules you already tested. Never put logic in routes.
6. **Never commit** `.venv/`, `*.db`, `__pycache__/`, or generated large files.
7. **Document as you go** — a docstring or a README paragraph per feature. Do not
   save documentation for the last day; the JD explicitly asks for it.
8. **Timebox.** Each day has a **Definition of Done**. If a task blows past its
   budget, apply the Cut Lines in Section 10 and move on.
9. **Keep a `docs/interview-notes.md`** where you jot the "why" behind decisions
   while they are fresh — you will need them in the interview.
10. **No scope creep.** New ideas go into Section 10 (stretch), not into today.

Suggested daily rhythm: **45 min plan/design → 3–4 h build+test → 30 min commit &
write up what you did and why.** Budget ~5 focused hours/day.

---

## 7. Seven-day schedule

### Day 1 — Wed Oct 14 · Foundation & data
- **Goal:** repo exists, package importable, you have realistic data.
- **Do:** `git init`; venv; `pyproject.toml`; package skeleton; `.gitignore`;
  `models.py`; `config.py`; **`synthetic.py`** fleet simulator
  (12 drives × 180 days, seeded, with distinct archetypes:
  *healthy, aging, wear-heavy, overheating, media-error-degrading, failing*).
- **DoD:** `python -m ssd_insight.cli generate --out sample_data/telemetry_sample.csv`
  produces a CSV you can open; `git log` shows ≥1 commit.

### Day 2 — Thu Oct 15 · Ingest & storage
- **Goal:** data flows from a file into a queryable database.
- **Do:** `db.py` (schema + indices, `INSERT OR REPLACE`, drive aggregate refresh);
  `parsers.py` (CSV, JSONL, smartctl text); CLI `ingest`, `list`.
- **Tests:** `test_parsers.py`, `test_db.py`.
- **DoD:** ingest the sample CSV; `ssd-insight list` shows all drives with
  first/last seen + sample counts; `pytest -q` green.

### Day 3 — Fri Oct 16 · Analytics engine (the interesting part)
- **Goal:** explainable health + forecasting.
- **Do:** `analytics.py` — `health_score()` with penalty breakdown, `wear_rate()`
  and `project_endurance()` via least squares, `temperature_stats()`,
  `error_growth()`. CLI `health`, `summary`.
- **Tests:** `test_analytics.py` with hand-checked numbers + edge cases
  (flat data → infinite/no projection; empty history).
- **DoD:** `ssd-insight health <drive_id>` prints score, grade, penalty breakdown,
  and "projected X days to endurance threshold".

### Day 4 — Sat Oct 17 · REST API + CLI polish  ← **MVP milestone**
- **Goal:** everything reachable over HTTP and through the CLI.
- **Do:** `api/main.py` (FastAPI): `GET /health`, `/drives`, `/drives/{id}`,
  `/drives/{id}/health`, `/drives/{id}/samples`, `/fleet/summary`,
  `/drives/{id}/anomalies`, `POST /ingest`. CLI `anomalies`, `report`.
  Auto OpenAPI at `/docs`.
- **Tests:** `test_api.py` with `fastapi.testclient.TestClient` (use a temp DB).
- **DoD:** `uvicorn ssd_insight.api.main:app` runs; `/docs` loads; endpoints return
  real data; `pytest -q` green. **Tag it: `git tag v0.1.0`.**

### Day 5 — Sun Oct 18 · Anomaly detection (the AI/ML story)
- **Goal:** find abnormal drives and abnormal points, and explain them.
- **Do:** `anomalies.py` — robust z-score (median/MAD, so it resists outliers) for
  fleet outliers and rolling-window temporal anomalies; **Isolation Forest**
  (scikit-learn) over per-drive feature vectors; optional `try/except ImportError`
  so the project still runs without sklearn.
- **Tests:** `test_anomalies.py` — inject a known spike, assert it's flagged.
- **DoD:** `ssd-insight anomalies` ranks drives/points with a reason string
  ("temperature 3.9σ above its 7-day median"). This is your ML talking point.

### Day 6 — Mon Oct 19 · Tests, docs, packaging, polish
- **Goal:** it looks and reads like a real engineering repo.
- **Do:** raise test coverage; `report.py` Markdown fleet/drive reports;
  `README.md` (pitch, install, quickstart, example output, architecture diagram);
  `docs/architecture.md`; `Makefile`; **Dockerfile**; `.github/workflows/ci.yml`;
  `ruff` clean; add a `LICENSE` and a small committed `sample_data/`.
- **DoD:** fresh clone → `make install && make test && make demo` works. README is
  good enough that a stranger can run it in 5 minutes.

### Day 7 — Tue Oct 20 · Rehearse, do not build
- **Goal:** deliver a 5–8 minute demo fluently.
- **Do:** record a terminal demo (or prepare exact commands); write
  `docs/interview-notes.md`; dry-run the demo twice end-to-end; prepare answers to
  **Section 9**; prepare 3 questions to ask *them*. Freeze the code (no new
  features). Bug-fix only.
- **DoD:** you can demo without looking at notes, and every claim you make maps to
  a file/line you can show.

---

## 8. Deliverables checklist (what you show them)

- [ ] Public GitHub repo `ssd-insight` with a clean commit history
- [ ] `README.md` with pitch, quickstart, example output, architecture diagram
- [ ] Working CLI demo (ingest → summary → health → anomalies → report)
- [ ] Working REST API with browsable `/docs`
- [ ] Test suite (`pytest -q` green) + CI badge
- [ ] `docs/architecture.md` and `docs/interview-notes.md`
- [ ] Dockerfile (deploy story) + Makefile
- [ ] `sample_data/telemetry_sample.csv` (so reviewers can run it instantly)

---

## 9. Interview prep — demo script & likely questions

**5-minute demo script**
1. One-sentence pitch (Section 1).
2. Show `sample_data` generator → "I simulated a realistic fleet because I had no
   production logs; here are 6 archetypes."
3. `ssd-insight ingest` → `list` → `summary` (fleet at a glance).
4. `ssd-insight health <bad-drive>` → walk through penalty breakdown + forecast.
5. `ssd-insight anomalies` → "this drive is degrading; here's the root cause."
6. Open `/docs` → hit one endpoint live.
7. Close: architecture decisions + what you'd do next.

**Likely questions (prepare a 30-second answer for each)**
- *Why SQLite and not Postgres?* (Correct scope for a tool with local/batch data;
  the storage layer is isolated in `db.py` so swapping is easy.)
- *Why robust z-score instead of a normal one?* (Median/MAD resists outliers; a
  single failing drive would distort mean/std.)
- *Isolation Forest vs. your statistical method — when?* (Univariate/explainable
  vs. multivariate correlation; you use both, deliberately.)
- *How did you test the health score?* (Hand-computed fixtures + boundary values.)
- *Biggest bug you hit and how you found it?* (Have one real story ready — keep it.)
- *How would this scale to 10k drives?* (Batch ingest, indices, aggregate tables,
  time-series DB, sampling — talk trade-offs.)
- *What's your endurance-forecast limitation?* (Linear extrapolation ignores
  workload changes; real models need write-amplification/WA_F context.)

**Ask them:** how interns are mentored, what a "day on the team" looks like, which
subsystem (firmware/validation/tooling) the internship project lives in.

---

## 10. Cut lines, risks, and stretch goals

**If you fall behind, cut in this order (bottom = cut first):**
1. Web dashboard
2. Prometheus endpoint
3. Isolation Forest (keep statistical anomalies — still an ML story)
4. Dockerfile / CI
5. Markdown report rendering (show raw JSON instead)
6. smartctl text parser (keep CSV/JSONL)

**Never cut:** the health score, the endurance forecast, the API, and the tests.
Those are the four things that map most directly to the JD.

**Risks & mitigations**
- *No real data* → the synthetic generator is a feature, present it as one
  ("I designed realistic drive archetypes"). Commit the sample output.
- *Python 3.14 wheel issues* → core stays pure-stdlib; sklearn/numpy are optional.
- *Scope creep* → everything new goes to stretch; the MVP gates every day.
- *Demo breaks live* → pre-record a backup terminal capture on Day 7.

**Stretch goals (only after MVP + polish):** listed in Section 3.

---

## 11. Definition of success

By Tue night you can:
- `git clone` your repo, run it on Linux, and demo it in under 8 minutes.
- Point at any JD bullet and name the module that proves it.
- Explain every design decision, including the trade-offs you *didn't* take.

Good luck. Ship the MVP.
