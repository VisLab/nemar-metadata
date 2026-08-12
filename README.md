# nemar-metadata

[NEMAR](https://nemar.org) is a repository for open-access neuroimaging and behavioral data in [BIDS](https://bids.neuroimaging.io/index.html). 

This repository discovers, mirrors, and summarizes metadata for the [nemarDatasets](https://github.com/nemarDatasets) GitHub organization for HED
(Hierarchical Event Descriptors) annotation.

- dataset repositories are named **`nm******`** *or* **`on******`**. The `on` datasets were exported from the corresponding `ds` datasets on [OpenNeuro]()
- each dataset carries a **`.nemar/`** subdirectory whose contents are mirrored
  alongside the top-level files.
- All pipeline logic lives in the shared **`hed-metadata-toolkit`** package; this
repo is a thin consumer that supplies NEMAR-specific configuration on the
command line. 
- The `hed-*` commands are created by installing the toolkit — they
are not files in this repo. Every command also runs as
`python -m hed_metadata_toolkit.<module>`, and every command supports `--help`.

---

## Environment setup

```powershell
# from the repo root, in PowerShell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -e .                              # installs the toolkit + hed-* commands
pip install -e PATH\TO\hed-metadata-toolkit   # editable, so toolkit edits are picked up
```

Put a GitHub token in a `.env` file at the repo root (used by every networked
command):

```
GITHUB_TOKEN=ghp_your_token_here
```

Confirm the commands are installed:

```powershell
hed-fetch-repo-list --help
```

> **Run every command from the repository root.** Paths resolve relative to the
> current directory (`datasets/dataset_summaries/…`, `datasets/dataset_repos/…`).

> **NEMAR-critical:** every command that talks to GitHub **requires
> `--org nemarDatasets`** - there is no default organization, and a wrong org
> 404s per file and leaves empty dataset folders. `--prefix` already defaults
> to `nm on`, so it can be omitted; `--include-subdir .nemar` applies only to
> Step 2.

---

## Quick start — full pipeline

Run these in order from the repo root. Each step consumes the previous step's
output, so the order matters.

```powershell
# 1. List every repo in the org              -> datasets/dataset_summaries/datasets.tsv
hed-fetch-repo-list --org nemarDatasets

# 2. Build the file listing for nm* repos,    -> datasets/dataset_summaries/repo_contents.json
#    including each repo's .nemar/ contents
hed-sync-repo-contents --org nemarDatasets --prefix nm --prefix on --include-subdir .nemar

# 3. Download the listed files locally         -> datasets/dataset_repos/<nm******>/...
hed-sync-local-files --org nemarDatasets

# 4. (optional) Download per-participant event files
hed-sync-repo-file-contents --org nemarDatasets

# 4b. (optional) Just LIST event files (no download)  -> datasets/dataset_summaries/event_files.{json,tsv}
hed-list-event-files --org nemarDatasets --prefix nm --prefix on

# 5-7. Build / enrich / sort the dataset summary
hed-extract-summary-info
hed-update-summary
hed-sort-datasets
```

To verify Step 3 actually downloaded the `.nemar` files:

```powershell
Get-Content datasets\dataset_repos\nm000105\.nemar\metadata.json
```

---

## Steps in detail

### Step 1 — Fetch the repository list

**Command:** `hed-fetch-repo-list`

Lists every repository in the organization and writes their names and
`updated_at` timestamps.

```powershell
hed-fetch-repo-list --org nemarDatasets
```

| Option | Default | Purpose |
|--------|---------|---------|
| `--org` | *(required)* | **Set to `nemarDatasets`.** |
| `--output` | `datasets/dataset_summaries/datasets.tsv` | Destination TSV. |
| `--token` | `$GITHUB_TOKEN` | GitHub PAT. |

**Output:** `datasets/dataset_summaries/datasets.tsv` (all org repos, incl.
non-dataset ones like `.github` — those are filtered out in Step 2, which keeps
only `nm*` and `on*`).

### Step 2 — Sync the file listing (incl. `.nemar/`)

**Command:** `hed-sync-repo-contents`

Reads `datasets.tsv`, keeps `nm******` and `on******` repos, and makes one
recursive git-tree call per repo to derive its `subjects`, `datatypes`,
`event_files`, and `top_level_files` (the latter includes the `.nemar/`
contents). This produces the *listing/metadata only* — it does not download file
contents (Step 3 does).

```powershell
hed-sync-repo-contents --org nemarDatasets --prefix nm --prefix on --include-subdir .nemar
```

| Option | Default | Purpose |
|--------|---------|---------|
| `--org` | *(required)* | **Set to `nemarDatasets`.** |
| `--prefix` | `nm on` | Repeatable; the default already matches NEMAR (`''` = all). |
| `--include-subdir` | *(none)* | **Set to `.nemar`** to include `.nemar/<file>` entries (repeatable). |
| `--force` | off | Re-fetch all repos, ignoring `synced_at`. |
| `--retry-failed` | off | Re-attempt repos in `repo_contents_failures.json`. |
| `--repo nm000105` | — | Process a single repo (handy for testing). |
| `--tsv` / `--out` | under `datasets/dataset_summaries/` | Override input/output paths. |

**Output:** `datasets/dataset_summaries/repo_contents.json` — per repo:
`synced_at`, `updated_at`, `truncated`, `top_level_files` (incl. `.nemar/<name>`
blobs), `subjects`, `datatypes`, and `event_files`.

### Step 3 — Download files locally

**Command:** `hed-sync-local-files`

Creates `datasets/dataset_repos/<repo>/` and downloads every blob in
`repo_contents.json` (including the `.nemar/<file>` entries; the `.nemar/`
subdirectory is recreated automatically). SHA-based incremental skipping means
re-runs only fetch changed/new files.

```powershell
hed-sync-local-files --org nemarDatasets
```

| Option | Default | Purpose |
|--------|---------|---------|
| `--org` | *(required)* | **Set to `nemarDatasets`** (otherwise every file 404s). |
| `--retry-failed` | off | Re-attempt files in `download_failures.json`. |
| `--force` | off | Re-download even when the SHA matches. |
| `--workers N` | 10 | Parallel download threads. |
| `--max-size BYTES` | 524288 | Skip blobs larger than this. |
| `--repo nm000105` | — | Sync a single repo. |
| `--contents` / `--datasets` | under `datasets/` | Override input listing / output root. |

**Output:** files under `datasets/dataset_repos/<repo>/` (incl. `.nemar/`);
failures recorded in `datasets/dataset_summaries/download_failures.json`.

> **Re-running after a failed/partial run:** files already recorded in
> `download_failures.json` are *skipped* on a normal re-run. Add `--retry-failed`
> to retry them (or `--force` to re-fetch everything). Example, to recover after
> a run that used the wrong org:
> `hed-sync-local-files --org nemarDatasets --retry-failed`

### Step 4 — Sync per-participant event files *(optional)*

**Command:** `hed-sync-repo-file-contents`

For each repo, reads `participants.tsv` to find the first participant and
downloads its `*_events.tsv` / `*_events.json` files.

```powershell
hed-sync-repo-file-contents --org nemarDatasets
```

Same `--org` requirement and `--retry-failed` / `--force` / `--workers` options
as Step 3.

### Steps 5–7 — Dataset summary

These read the local data and the listing; they make no network calls, so they
take no `--org`.

```powershell
hed-extract-summary-info     # -> datasets/dataset_summaries/dataset_summary.tsv
hed-update-summary           # -> dataset_summary_updated.tsv (titles, HED versions, link counts)
hed-sort-datasets            # -> dataset_summary_sorted.tsv
```

Optional README corpus. `dataset_repos/` holds only `nm*`/`on*` dataset folders,
so just run it with no `--dirprefix` (it takes a single prefix, which can't match
both `nm` and `on`):

```powershell
hed-extract-readme-summaries
```

Citation steps (`hed-collect-citations`, `hed-assign-citation-ids`,
`hed-enrich-pub-ids`, …) are available from the toolkit if/when NEMAR citation
curation is needed; add a `config/citation_skip_list.txt` first.

### Optional — list event files without downloading

**Command:** `hed-list-event-files`

Builds a manifest of every BIDS event file (`*_events.tsv` / `*_events.json`)
that sits at the repo root or **anywhere beneath a top-level `sub-*/`
directory** — event files under `derivatives/`, `sourcedata/`, etc. are
ignored. It only reads the recursive git-tree per repo; **no file contents are
downloaded.**

```powershell
hed-list-event-files --org nemarDatasets --prefix nm --prefix on
```

| Option | Default | Purpose |
|--------|---------|---------|
| `--org` | *(required)* | **Set to `nemarDatasets`.** |
| `--prefix` | `nm on` | Repeatable; the default already matches NEMAR. |
| `--force` | off | Re-list every repo (otherwise unchanged repos are skipped). |
| `--out` / `--tsv-out` | under `datasets/dataset_summaries/` | Override the JSON manifest / flat TSV paths. |

**Outputs:** `datasets/dataset_summaries/event_files.json` — per repo:
`synced_at`, `updated_at`, and `event_files` as a list of `{path, size, sha}`
objects (`size`/`sha` from the git-trees blob entry) — and `event_files.tsv`
(`repo`, `path`, `size`, `sha`). Incremental: on re-run, a repo is re-listed
only when its `datasets.tsv` `updated_at` is newer than the stored `synced_at`.

---

## Layout

```
nemar-metadata/
├── datasets/
│   ├── dataset_repos/          # locally mirrored files per nm*/on* dataset (incl. .nemar/)
│   └── dataset_summaries/      # datasets.tsv, repo_contents.json, summary TSVs
└── pyproject.toml              # dependency pins only — this repo has no code
```
