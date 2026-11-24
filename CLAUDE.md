# CLAUDE.md

BT3041 team project: predicting COVID-19 severity from nasal scRNA-seq (Ziegler et al. 2021). Five people, each with their own Claude sessions.

**Before doing anything else in a session:**
1. Read [CONVENTIONS.md](CONVENTIONS.md) (rules; §1 is for you, the agent) and [CONTEXT.md](CONTEXT.md) (project state, dataset facts, labels, **data contracts**).
2. Ask the human **which teammate you are working for and which workstream** (codes and owners: [TEAM_PLAN.md](TEAM_PLAN.md) §3), unless they already said. Stay inside that workstream's files.
3. Run `git branch --show-current`. If it is `main`, stop and ask the human to create a branch (`<name>/<short-task>`).

**Hard rules (details in CONVENTIONS.md §1):** no `git commit/push/merge/rebase/reset --hard/stash drop`, no PRs; never touch `data/raw/`; never commit data; ask before installing packages, deleting files or editing shared files; report only what you actually ran and saw.

Use the project venv: `.venv/Scripts/python` (Windows) or `.venv/bin/python` (macOS/Linux). Run pipeline steps as `python scripts/NN_*.py` from the repo root.

**End of every session:** write `logs/YYYY-MM-DD_HHMM_<name>_<workstream>.md` from [logs/_TEMPLATE.md](logs/_TEMPLATE.md). Remind the human to run `/export` and save the transcript to `logs/transcripts/` with the same filename stem, and to update their row in CONTEXT.md §8.
