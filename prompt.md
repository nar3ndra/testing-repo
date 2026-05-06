Create a README.md for the `sas_splitter` module in this repo.

## Discovery (do this first, do not skip)
1. Locate the module directory and list its files. Read every source file end-to-end before writing.
2. Read the module's `SKILL.md` (if present), any `__init__.py`, entry point, and CLI/agent invocation surface.
3. Identify: public API, inputs (file types, expected SAS constructs), outputs (chunk format, metadata, file layout on disk), config knobs (env vars, CLI flags, kwargs), and dependencies on sibling agents (converter, aggregator, reconciliation).
4. Trace one real example through the code — pick an actual SAS file from the repo if available — and capture the resulting split structure.
5. Match the documentation tone and structure of existing READMEs in this repo. If a sibling module has a README, mirror its section order and heading style. Do not invent a new format.

## Required sections (in this order)
1. **Overview** — one paragraph: what it splits, why splitting is needed in the SAS-to-PySpark pipeline, where it sits relative to the other agents.
2. **How it works** — the splitting strategy (boundary detection: PROC/DATA steps, macro definitions, %INCLUDE handling, etc.). Cite the actual functions/classes by name.
3. **Inputs & Outputs** — exact input contract and the on-disk output layout (folder hash convention, timestamped run names if applicable, manifest/metadata files).
4. **Usage** — minimal runnable example (Python API + CLI/agent invocation if both exist). Real commands, no pseudocode.
5. **Configuration** — env vars, `.env` keys, skill-file hardcoded values, and any defaults. Reference the actual variable names from the code.
6. **Integration** — how the converter and aggregator agents consume splitter output. One short diagram in mermaid is fine if it clarifies the handoff.
7. **Limitations & known edge cases** — only ones grounded in the code (e.g., comment handling, nested macros, encoding). Do not speculate.

## Constraints
- No filler, no marketing voice. Dense and technical.
- Every claim must be traceable to code you actually read. If something isn't in the code, leave it out.
- Use fenced code blocks with correct language tags.
- If you find ambiguity (e.g., two plausible split strategies in different code paths), surface it as an open question at the bottom rather than guessing.
- Write to `<module_path>/README.md`. Do not modify source files.

When done, print a short summary of what you documented and flag anything that looked broken or inconsistent during discovery.
