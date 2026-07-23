# Nemar-metadata developer instructions

> **Local environment**: If `.status/local-environment.md` exists in the repository root, read it first — it contains machine-specific shell, OS, and venv details (e.g. Windows/PowerShell vs Linux/bash) that override the generic commands shown here.

## Code style

- Google-style docstrings; use `Parameters:` not `Args:`
- Line length: 120 characters (configured in `pyproject.toml`)
- Markdown headers use sentence case: capitalize only the first word (and proper nouns/acronyms)
- When creating work summaries, place them in `.status/` at the repository root

## Project overview

This repository (`nemar-metadata`) downloads and extracts metadata files from repositories in the
[nemarDatasets](https://github.com/nemarDatasets) GitHub organization. The extracted metadata supports
analysis and reporting on NEMAR (Neuro EEG Modalities Archive) datasets.

NEMAR is a data archive for EEG and related neuroimaging datasets following BIDS conventions.
Each dataset is stored as a separate `nm*` repository in the nemarDatasets GitHub organization.

## Project structure

```
src/
    create_repo_list.py     # fetches repo list from nemarDatasets org → datasets/datasets.tsv
    sync_repo_contents.py   # fetches top-level file listings for each repo → datasets/repo_contents.json
    sync_local_files.py     # downloads top-level blobs from each repo → datasets/<repo>/
datasets/                   # downloaded data (gitignored)
results/                    # analysis outputs (gitignored)
tests/                      # unit tests
```

## Data pipeline

The three scripts run in sequence to populate `datasets/`.

### Step 1: create_repo_list.py

Calls the GitHub REST API (`/orgs/nemarDatasets/repos`) to list every repository in the
organization and writes a TSV:

```
datasets/datasets.tsv   columns: name, updated_at
```

### Step 2: sync_repo_contents.py

Uses the GitHub GraphQL API to batch-fetch the top-level file/directory listing for
every `nm*` repository (10 repos per request). Writes:

```
datasets/repo_contents.json           { "<repo>": { "synced_at": ..., "entries": [...] } }
datasets/repo_contents_failures.json  repos whose entries came back empty
```

Incremental mode: repos whose `synced_at >= updated_at` are skipped automatically.

### Step 3: sync_local_files.py

Downloads every top-level blob for each `nm*` repo into `datasets/<repo>/`:

- SHA-based incremental skip avoids re-downloading unchanged files
- Parallel downloads via `ThreadPoolExecutor` (default 10 workers)
- Configurable max file size (default 512 KB) to skip large binaries
- Failure tracking in `datasets/download_failures.json`

## Key constants

Each script declares the GitHub organization at the top:

```python
ORGANIZATION = "nemarDatasets"
```

This is the only value that needs updating if the target organization changes.

## Development environment

### Setup

Always use the `.venv` virtual environment — never the system Python. Activate it first, then
run commands, or invoke `.venv\Scripts\python.exe` directly. See `.status/local-environment.md`
for OS-specific activation syntax.

```powershell
.venv\Scripts\Activate.ps1
.venv\Scripts\python.exe -m pip install -e ".[dev]"
```

A `GITHUB_TOKEN` in a `.env` file (or environment variable) is required for all three
scripts. Unauthenticated GraphQL requests are not supported; REST requests are heavily
rate-limited without a token.

### Running the pipeline

```powershell
python src/create_repo_list.py
python src/sync_repo_contents.py [--force] [--retry-failed] [--repo ds000001]
python src/sync_local_files.py   [--repo ds000001] [--workers 10] [--force]
```

### Testing

```powershell
# All tests
python -m pytest tests/ -v

# Single module
python -m pytest tests/test_create_repo_list.py -v
```

### Linting and formatting

```powershell
# Check
ruff check src/ tests/
ruff format --check src/ tests/

# Auto-fix
ruff check --fix --unsafe-fixes src/ tests/
ruff format src/ tests/
```

## Common pitfalls

- Always set `GITHUB_TOKEN` — unauthenticated GraphQL requests are not supported
- `datasets/` and `results/` are gitignored; never commit downloaded data
- Do not use `&&` for command chaining in PowerShell — use `;` instead
- Always activate the virtual environment before running Python/pip commands
- Check `.status/local-environment.md` for shell-specific command syntax
