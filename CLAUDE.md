# CLAUDE.md — Project conventions for Claude sessions

Read this first. It describes what this repo is and the things that matter for
working in it. See `README.md` for the full pipeline/how-to-run.

## What this repo is

A thin consumer of the shared **`hed-metadata-toolkit`** package, scoped to the
[nemarDatasets](https://github.com/nemarDatasets) GitHub organization. It mirrors
and summarizes per-dataset metadata for HED annotation. NEMAR is structured like
OpenNeuro (one repo per dataset); the differences are encoded as command-line
configuration, not forked code:

- dataset repos are named **`nm******`** and **`on******`** (both are NEMAR
  datasets) → `--prefix nm --prefix on` (the `--prefix` flag is repeatable)
- each dataset has a **`.nemar/`** subdirectory to mirror → `--include-subdir .nemar`
- organization → `--org nemarDatasets`

All pipeline logic is in the toolkit; this repo holds configuration, cached
data under `datasets/`, and notes. There are **no repo-specific pipeline
scripts** under `src/`.

## Toolkit dependency

`pyproject.toml` pins the toolkit by **local path**
(`hed-metadata-toolkit @ file:///H:/Repos/hed-metadata-toolkit`). `pip install -e .`
installs it and puts the `hed-*` commands on PATH. For live toolkit edits:
`pip install -e H:/Repos/hed-metadata-toolkit`. The `file://` pin will not
resolve on a CI runner — switch to a published/Git-URL pin before enabling CI.

If a needed behavior differs from OpenNeuro, prefer adding a **configuration
option to the toolkit** (like `--prefix` / `--include-subdir`) over forking code
into this repo.

## Layout

Toolkit commands use the OpenNeuro-style layout, run from the repo root:
`datasets/dataset_summaries/` (datasets.tsv, repo_contents.json, summary TSVs)
and `datasets/dataset_repos/<nm******|on******>/` (mirrored files, including `.nemar/`).

## Windows / Cowork sandbox quirks

- Use the **Read tool** (Windows path) for current file contents; the bash
  Linux mount can serve a **stale/truncated** snapshot of just-written files.
- Use **bash** (Linux mount) for running code, `ls`, `wc`, `git`.
- The user runs Python/tests locally in `.venv` (PowerShell) and pastes output;
  don't treat sandbox runs as authoritative.

## Session reporting

Every session that writes/moves/deletes files produces a dated note in
`.status/` (gitignored): `.status/session_YYYY-MM-DD_<topic>.md`. Plans also go
in `.status/`. `.status/temp_files/` holds retired files kept for reference.
