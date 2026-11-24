# CONVENTIONS: ground rules for humans and agents

Short and strict. If a rule blocks you, ask the team; don't work around it. Project state and data contracts are in [CONTEXT.md](CONTEXT.md); who does what and when is in [TEAM_PLAN.md](TEAM_PLAN.md).

---

## 1. Rules for AI agents (Claude Code or any other)

1. **Never** run `git commit`, `git push`, `git merge`, `git rebase`, `git reset --hard`, `git stash drop`, `git push --force`, `git tag`, or `git branch -D`. Never open, approve or merge PRs. Never run anything else that alters history or a remote. **A human reviews the diff, commits, pushes and merges.**
2. Read-only git is fine: `git status`, `git diff`, `git log`, `git branch`, `git show`. Create or switch branches (`git switch -c <name>/<task>`) **only when the human asks**.
3. **Before editing anything, check the branch** (`git branch --show-current`). If it is `main`, stop and ask the human to create a branch.
4. Identify **which teammate you work for and which workstream** at the start (see the table in TEAM_PLAN.md). Stay inside that workstream's files: its numbered scripts, its notebooks, `results/<workstream>/`, and its own row in CONTEXT.md §8. Touch nothing else unless the human says so.
5. **Never modify or delete anything in `data/raw/`.** Never commit or stage data or large files (> 50 MB, or any `.h5ad`). Never overwrite another workstream's files or outputs.
6. **Ask the human first** before: installing or upgrading packages; deleting any file; editing shared files (§3); downloading data; running anything expected to take > 15 minutes.
7. **Don't fabricate.** Report only what you actually ran and saw. Every number in a doc, slide, log or PR must come from an output file or console output in this session. If something wasn't run, say "not run".
8. **Write the session log** (§7) before the session ends, even if the session failed.
9. Write plain student code (§6). This code is graded.

## 2. Git flow (humans)

1. **Start of every session:** `git switch main`, `git pull`, then `git switch -c <name>/<short-task>` (e.g. `vijay/qc`, `ruchir/classifier-v1`). For an existing branch: `git switch <branch>` then `git merge main`.
2. Small, focused commits **made by the human** after reading `git diff`.
3. Push the branch and open a **PR into `main`**. Fill in the PR template: what changed, the command(s) that regenerate results, links to the session log(s).
4. **At least one teammate reviews.** For a result PR, the reviewer reruns the regenerate command on their machine and checks the numbers match. A human merges, then deletes the branch.
5. **Nobody pushes directly to `main`.**
6. **Shared-file changes** (§3) go in **their own small PR**, separate from analysis work, so conflicts stay small and everyone notices them.
7. Pull `main` often. Contract files may have changed.

## 3. Shared files (change only via their own small PR, ask the team)

`CONTEXT.md` §1–7 (problem, facts, labels, decisions, **contracts**, definitions) · `CONVENTIONS.md` · `CLAUDE.md` · `TEAM_PLAN.md` (owner: Sidharth) · `requirements.txt` · `scripts/common.py` · `scripts/classify_utils.py` (owner: Ruchir; others import it but don't edit it) · `.gitignore` · `.github/` · `data/external/` (owner: Eswar).
Exception: each person updates **their own row** in CONTEXT.md §8 (Current status) in their normal PR.

## 4. Reproducibility (there is no shared environment)

- **Python 3.12**, each person's own venv in `.venv/` (gitignored). Install with `pip install -r requirements.txt`. Adding a package = a PR that pins it in `requirements.txt`.
- **From a clean clone + `data/raw/` + the venv, every result must regenerate** by running `python scripts/NN_*.py` in numeric order (pipeline table: CONTEXT.md §6).
- **Contract files are produced only by numbered scripts**, not notebooks. Notebooks are for exploration and figures, read contract files, and must **run top-to-bottom on a fresh kernel** ("Restart & Run All") before committing.
- **Relative paths only**, via `scripts/common.py` (`common.ROOT`, `common.QC_H5AD`, …). No `C:\Users\...`.
- **Fixed seeds:** `common.SEED` (42) for UMAP, Leiden, RF, permutations; CV repeats use seeds 0–9. Pass `random_state` explicitly.
- `data/processed/` is **gitignored and rebuilt**, never sent by hand (no WhatsApp/Drive copies of processed files).
- Outputs go to `results/<workstream>/` (tables `.tsv`, figures in `results/<workstream>/figures/`). Small tables and figures **are committed** so the deck can use them. Nothing > 10 MB in `results/`.
- **A PR that adds a result must state the exact command(s) that regenerate it**, and the reviewer reruns them.
- Gene sets / external tables: download once, commit into `data/external/` with a line in `data/external/SOURCES.md` (URL, version, date, md5). No live API calls inside the pipeline.

## 5. Statistical hygiene

- **The patient (`donor_id`) is the sample unit.** Splits, tests and p-values are at patient level. No cell-level CV, no cell-level DE p-values as findings.
- Everything learned from data (scaling, HVGs, PCA, feature selection, imputation, hyperparameters) is fit **inside the training fold only**. **No test-set peeking**, including looking at test labels when choosing features or thresholds.
- Report **BH-FDR-corrected** p-values (q-values) for any family of tests, and say what the family is.
- Classification: report AUROC **and** balanced accuracy, with spread over CV repeats and a **label-permutation p-value**. Never accuracy alone.
- **State n (patients) on every plot** (in the title or legend, per group) and in every table that reports a test.
- Labels come from `common.GROUP_MAP` only. No ad-hoc relabelling.

## 6. Code style

- Plain, readable student code: simple functions and loops, clear variable names, a short comment where the *why* isn't obvious. A docstring at the top of each script saying what it reads and writes.
- No classes, frameworks, config systems or clever one-liners unless there is a real need. Repeating three lines is better than a premature abstraction.
- Each script runs end-to-end with `python scripts/NN_name.py` and prints what it wrote.

## 7. Session logs (the course's required AI-usage record)

- Every agent session ends with a log file `logs/YYYY-MM-DD_HHMM_<name>_<workstream>.md`, using [logs/_TEMPLATE.md](logs/_TEMPLATE.md). Time is the session **start**, 24 h, IST. `<name>` is lowercase first name.
- The human then runs `/export` in Claude Code and saves the full transcript as `logs/transcripts/<same stem>.txt`.
- Both are committed with the work and linked in the PR. Don't edit old logs, except to fix a factual error (say so in the file).

## 8. Naming

| Thing | Pattern | Example |
|---|---|---|
| Branch | `<name>/<short-task>` | `sahaj/patient-features` |
| Pipeline script | `scripts/NN_<topic>.py` (NN from CONTEXT.md §6) | `scripts/05_classify.py` |
| Helper module | `scripts/<topic>_utils.py` (no number, importable) | `scripts/classify_utils.py` |
| Notebook | `notebooks/NN_<topic>.ipynb` (NN = the step it belongs to) | `notebooks/02_umap_figures.ipynb` |
| Scratch notebook | `notebooks/scratch/<name>_<topic>.ipynb` (gitignored) | |
| Figure | `results/<ws>/figures/NN_<what>.png` (300 dpi; `.pdf` optional) | `results/classify/figures/05_roc_comp_vs_pb.png` |
| Table | `results/<ws>/<what>.tsv` (tab-separated, header row) | `results/classify/cv_metrics.tsv` |
| Cell-type slug | lowercase, spaces → `_` | `developing_ciliated_cells` |
| Session log | `logs/YYYY-MM-DD_HHMM_<name>_<ws>.md` | `logs/2026-10-31_0930_vijay_qc.md` |

Workstream codes (`<ws>`): `qc, composition, classify, isg, de, attribution, sensitivity, perturb, targets, validation, lit, deck, report, setup`.
