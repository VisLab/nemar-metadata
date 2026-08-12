# nemar-metadata

**`AGENTS.md` at the repository root is the instruction set for this project. Read it before answering, and follow it.** It covers the organization, the commands and their required flags, the data files and which are curated, the toolkit dependency, the conventions, and the working agreements.

This file is a pointer and duplicates nothing. One source, several pointers: a rule stated in two files is a rule that will disagree with itself.

Machine-specific facts - interpreter, local paths, the shared cache location - are in `.status/local-environment.md`. That file is gitignored, so it is absent from clones and from CI; read it when it is there and ignore its absence when it is not. Apart from the `pyproject.toml` toolkit pin, no committed file in this repository may contain a local path or a drive letter.
