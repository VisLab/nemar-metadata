# src

This directory holds no repo-specific pipeline scripts.

All discovery / summary / citation functionality lives in the
`hed-metadata-toolkit` package and is run via its `hed-*` console commands (or
`python -m hed_metadata_toolkit.<module>`). See the top-level
[`README.md`](../README.md) for the NEMAR workflow and the exact commands
(`--org nemarDatasets --prefix nm --prefix on --include-subdir .nemar`).

The earlier hand-written NEMAR scripts (`create_repo_list.py`,
`sync_repo_contents.py`, `sync_local_files.py`) were superseded by the toolkit
and archived under `../.status/temp_files/`.
