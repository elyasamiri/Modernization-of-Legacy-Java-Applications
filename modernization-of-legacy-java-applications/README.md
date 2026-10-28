# Modernization of Legacy Java Applications

MSc Project - Software Development, IT Project 3154658A, University of Glasgow (2026).

A multi-agent LLM pipeline that automates the modernization of legacy Java/Spring Boot applications: framework migration, vulnerability remediation, and test generation. Every stage runs behind a build-verification gate, and any change that breaks the build or the existing test suite is automatically rolled back — so the pipeline can only ever move a codebase forward from a known-good state.

## What the pipeline does

Given a target Spring Boot application, the pipeline runs through the following stages end-to-end:

1. **Baseline capture** — runs a dependency-check vulnerability scan and the existing test suite to record the "before" state.
2. **Migration** — upgrades the app via [OpenRewrite](https://docs.openrewrite.org/) (Spring Boot 2 → 3 / Java 8 → 17), followed by a small set of generic and project-specific fixes for issues OpenRewrite doesn't catch, then a build-verification gate.
3. **Vulnerability remediation** — an LLM agent reads the dependency-check scan, prioritizes vulnerable dependencies by severity and fix confidence, proposes and applies version overrides, and verifies the build (and full test suite) still passes before keeping each fix. Failed fixes are automatically rolled back.
4. **Test generation** — a coverage-driven agent parses JaCoCo XML reports, ranks under-covered classes, and generates new JUnit 5 tests via an LLM. Generated tests are discarded if they don't pass verification (the same verify-before-commit pattern used across all three agents above).
5. **Final verification** — full test suite + coverage run, results written out for evaluation.

Results for each run go into `evaluation/<app_id>/` (vulnerability scans, coverage XML) and a stage-by-stage JSON report is written to `agents/reports/pipeline_report_<app_id>.json`.

## Live dashboard

A read-only instance of the evaluation dashboard is deployed at **[dashboard-deploy-2t30.onrender.com](https://dashboard-deploy-2t30.onrender.com)**.

It replays the `pipeline_report_<app_id>.json` files produced by local pipeline runs as an animated, stage-by-stage view, alongside the before/after vulnerability, test, and coverage stat cards — so results can be reviewed without a local setup. It does not run the pipeline itself; migration, remediation, and test generation still only run locally against a target application directory (see [Setup](#setup)).

> The dashboard is hosted on Render's free tier, so it spins down after periods of inactivity. The first load after a while idle will show Render's own "waking up" screen for 20–50 seconds before the dashboard appears — this is normal.

## Repository structure

```
agents/                        Pipeline source (Python)
  orchestrator.py               Runs the full pipeline end-to-end against one target app
  remediation_agent.py          Vulnerability remediation agent
  test_generation_agent.py      Coverage-driven test generation agent
  run_batch.py                  Clones and runs the pipeline across a batch of target apps
  verify_apps.py                Checks each app's pipeline report for dissertation-evidence-clean status
  generate_evaluation_report.py Builds a markdown before/after summary from scan + report data
  llm_cache.py                  Lightweight local cache for LLM responses, so re-running a stage during development doesn't repeat identical calls
  fix_app7_versions.py          One-off utility script used during the multi-app validation round
  patch_final_test_counts.py    One-off utility script used during the multi-app validation round
  app.py                        Flask backend for the web dashboard
  reports/                      Per-app pipeline_report_<id>.json snapshots
  templates/                    Dashboard HTML
  tests/                        Unit tests for the pipeline and dashboard code (test_unit.py) — separate from the target apps' own Java test suites
  requirements.txt              Dependencies for the deployed dashboard only (Flask + gunicorn) — see Setup
  Procfile                      Render start command for the dashboard

target-apps/               Cloned target applications used for the multi-app validation round
legacy-app/                 Reference target application: spring-petclinic-rest
legacy-app-baseline/        Untouched baseline copy of legacy-app, kept for comparison
legacy-app-fresh-run/       Clean re-run of the pipeline against an untouched baseline clone
legacy-app-demo-run/        Run used for the dashboard demo

evaluation/                 Before/after vulnerability scans, coverage reports, and evaluation summaries per app
  reference-run/             The reference evaluation run (legacy-app) used as dissertation evidence

dev-history/                Archived scratch scripts / notes from development
docs/                       (reserved for project documentation)
```

> **Known clutter:** there's currently a duplicate, empty `agents/agents/reports/` directory and a second copy of `target-apps/` nested inside `agents/` — both look like leftovers from earlier runs rather than anything intentionally used. Worth deleting before submission; see [Roadmap](#roadmap).

## Setup

Requires Java (Maven-based target apps), Python 3.11+, and an OpenRouter API key.

```
cd agents
python -m venv venv          # if not already created
venv\Scripts\activate         # Windows
```

`agents/requirements.txt` currently only covers what the deployed dashboard needs (`flask`, `gunicorn`). To run the pipeline itself (`orchestrator.py`, `remediation_agent.py`, `test_generation_agent.py`), also install:

```
pip install -r requirements.txt
pip install openai python-dotenv
```

> A single `requirements.txt` covering both the dashboard and the full pipeline is on the [Roadmap](#roadmap) — for now, the two install steps above are both needed to run everything locally.

Create a `.env` file at the **repository root** (one level above `agents/`) with:

```
NVD_API_KEY=<your NVD API key>
OPENROUTER_API_KEY=<your OpenRouter API key>
```

Both are read via `python-dotenv`, which searches upward from the current directory, so a `.env` at the repo root is found correctly whether a script is run from the root or from inside `agents/`.

## Usage

All commands below are run from inside `agents/`.

Run the full pipeline against the default reference app (`legacy-app`):

```
python orchestrator.py
```

Run against a specific target app:

```
python orchestrator.py ../target-apps/app7-h2crud
```

or via environment variable:

```
set TARGET_APP_DIR=../target-apps/app7-h2crud
python orchestrator.py
```

Run the pipeline across the full batch of target apps (app7–app13):

```
python run_batch.py
```

Check which apps have a clean, dissertation-evidence-ready pipeline run:

```
python verify_apps.py
```

Generate a markdown before/after evaluation summary:

```
python generate_evaluation_report.py
```

Launch the results dashboard locally:

```
python app.py
```

## Evaluation results

### Reference run — `legacy-app` (spring-petclinic-rest)

**Vulnerability remediation**

| Metric | Before | After |
|---|---|---|
| Vulnerable dependencies | 13 | 12 |
| Total CVEs | 170 | 78 |
| Critical severity | 28 | 15 |
| High severity | 55 | 26 |
| Medium severity | 74 | 31 |
| Low severity | 13 | 6 |

**Test suite**

| Metric | Before | After |
|---|---|---|
| Total tests | 171 | 196 |
| Failing tests | 0 | 0 |

**Code coverage (JaCoCo)**

| Metric | Before | After |
|---|---|---|
| Line coverage | 88.4% (1273/1440) | 91.0% (1290/1417) |
| Branch coverage | 71.1% (270/380) | 73.5% (275/374) |
| Instruction coverage | 88.9% (5308/5969) | 91.2% (5244/5751) |

The migration stage also reported an estimated 7h 49m of manual effort saved (OpenRewrite's own self-reported estimate), and every proposed fix in the remediation batch (5/5) and generated test (3/3) passed verification and was kept — none needed to be rolled back for this run.

### Multi-app validation round

The pipeline was additionally run against seven public GitHub tutorial Spring Boot apps (`app7`–`app13`) to test generalization beyond the reference app, with per-app before/after vulnerability scans and coverage reports in `evaluation/`. Baseline-only scans are also recorded for `app2` (spring-petclinic), `app3` (BankApplicationBackend), and `app4` (Contrast Security's intentionally vulnerable Spring Boot app).

## Roadmap

Known gaps and next steps, based on the current state of the repo:

- Merge `requirements.txt` into a single file covering both the dashboard's Render deployment and the full pipeline (`openai`, `python-dotenv` are currently installed separately — see [Setup](#setup)).
- Remove the stray empty `agents/agents/reports/` directory and the duplicate `target-apps/` nested inside `agents/` (see [Repository structure](#repository-structure)).
- Run `app2` (spring-petclinic), `app3` (BankApplicationBackend), and `app4` (Contrast Security vulnerable app) through the full pipeline — currently only baseline vulnerability scans exist for these; no migration/remediation/test-generation pass has been recorded yet.
- Populate `docs/` with project documentation (currently empty).
- Write up final dissertation evidence from the multi-app validation round.

## Status

Actively developed as part of an ongoing MSc dissertation, from the initial baseline capture in July 2026 through ongoing refinement of the pipeline and dashboard. See the git log for the full development timeline, including `dev-history` for archived scratch work.

## Support

This is a solo MSc dissertation project. For questions about the project, contact the author directly at `3154658a@student.gla.ac.uk`, or open an issue on the [GitLab repo](https://stgit.dcs.gla.ac.uk/msc-project-for-information-technology/2025/it-project-3154658a/modernization-of-legacy-java-applications/-/issues).

## Contributing

This repository is submitted coursework for an individual MSc dissertation and is not open for external contributions. If you're the module supervisor or a marker reviewing this work, see the git log for the full development timeline, and `evaluation/` for the underlying before/after data behind the results reported above.

## Authors and acknowledgment

**Mohammad Elyas Amiri** — `3154658a@student.gla.ac.uk`
MSc Software Development, University of Glasgow — IT Project 3154658A (2026)

## License

This is an academic dissertation project submitted to the University of Glasgow; no open-source license has been applied. Usage and distribution are subject to the University's academic policies on student work. If you'd like the code released under an open-source license (e.g. MIT, Apache 2.0), add a `LICENSE` file and note it here — check with your supervisor first, since IP terms for dissertation code can be governed by university policy.
