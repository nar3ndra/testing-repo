---
name: fastapi-sas-orchestrator-designer
description: Design (do NOT implement) the production-grade FastAPI v0 backend that orchestrates the SAS-to-PySpark conversion pipeline for Jira ticket AKIT-433. Use this skill whenever the task is to design the POST /v0/start entry point, the orchestration layer over the sas_splitter / sas_executor / sas_converter modules, project structure, API contract, or file-handling strategy for the conversion backend — even if the user only says "design the conversion API" or "plan the orchestrator". Pairs with the fastapi-sas-orchestrator-implementer skill, which handles the build phase.
---

# FastAPI SAS-to-PySpark Orchestrator — Design Phase (Staff-Level)

You are the staff engineer designing the single entry-point backend for ticket **AKIT-433**.
Your job in this skill is to produce a **complete design only**. Do not write implementation
code. End with an approval gate.

> . The repository is your single source of truth. The "Module Hints"
> section below is only a rough orientation — it tells you what
> the modules roughly do and what to look for. It is NOT authoritative and may be stale or
> wrong. You must read the actual module code and design against what you find there.

## The Ticket (AKIT-433)

We have four conversion steps already built: **splitting, executing, converting, aggregating**.
They are currently invoked manually one at a time. We need a single FastAPI entry point that
launches the whole pipeline in sequence, callable from a future UI and from `curl`.

- Endpoint: `POST /v0/start`
- Input: a path to a SAS file — in a **Databricks Volume** (production) or a **local path**
  (local dev/testing, download skipped).
- Output: success JSON with a generated `run_id` (UUID).
- Acceptance criteria: a `curl` call runs all steps in order and produces **output files
  identical to manual execution**.

## Step 0 — Get the codebase research (do this before designing anything)

You cannot design a correct orchestrator over modules you have not researched. The
research is captured in a handoff file so it is done once and reused by the
implementation phase:

1. **Check for `.claude/handoff/akit-433-research.md`.**
   - If it exists and is current, read it in full — that IS your research result. Do not
     re-read the whole codebase.
   - If it is missing or stale, **delegate research to the `sas-codebase-researcher`
     subagent** (Task tool). It reads all three modules end to end and writes the handoff
     file. If subagents are unavailable, do the research inline and write the same file
     yourself, following the structure that subagent defines.
2. **Confirm the handoff file covers**: real signatures of `sas_splitter`,
   `sas_executor`, `sas_converter`; how each step is invoked manually today; the on-disk
   data flow and filename conventions; the converter integration verdict (importable
   callable vs argparse-only); the dependency inventory (pydantic v1 vs v2 especially);
   and repo structure/style. If anything is missing, send the researcher back for it.

**Base every design decision on the handoff file**, citing what the code actually shows.
Do not design from the "Module Hints" section — those are orientation only.

## Critical Reframe: 4 Ticket Steps → 3 Modules

The ticket says "4 steps" but the codebase exposes **three modules**. The converter module
already combines *converting* and *aggregating*. Confirm this against the real code, then
design the orchestrator around three module calls:

| Ticket step | Module | What to confirm in the code |
|-------------|--------|------------------------------|
| 1. Splitting | `sas_splitter` | the single-file split entry point + what it writes |
| 2. Executing | `sas_executor` | the file-execution entry point + its result object |
| 3. Convert + 4. Aggregate | `sas_converter` | that one run does convert AND aggregate |

If the real code contradicts this mapping, follow the code and flag the discrepancy.

## Module Hints (orientation only — verify everything against the real source)

These are rough notes from external screenshots. Use them only to know where to look.
The actual repository overrides anything here.

- **`sas_splitter`** — likely `ai_orchestrator.sas_splitter.splitter` exposing something
  like `split_sas_file(...)`, a `SASCodeSplitter` class, and a `BatchSplitter`. Splitting a
  file appears to write a `*_split.sas` and a `*_metadata.json`, and to inject Snowflake
  checkpoint macros. Confirm the real function name, signature, output filenames, and
  whether it also emits a `checkpoint_config.yaml`.
- **`sas_executor`** — likely `ai_orchestrator.sas_executor.file_executor` exposing
  something like `run_sas_file(...)`, plus an `ssh_runner`. It appears to run a `.sas`
  file on a remote SAS server over SSH and return a result object with success/log/exit
  fields. Executing the checkpoint-injected file appears to populate Snowflake checkpoint
  tables that the converter later reconciles against — confirm this ordering dependency.
- **`sas_converter`** — appears to be driven by `run_batch_and_aggregate.py`, which seems
  to convert all chunks and aggregate them in one run, reading a folder containing the
  splitter outputs. Confirm whether it has an importable callable or is argparse-only,
  and what input files it requires.

## Deliverables (produce all, in order — grounded in your Step 0 research)

### Step 1 — High-level architecture
- Proposed project folder structure (routers, schemas, services, utils, core/config),
  extending whatever the repo already has.
- The orchestration layer: a `ConversionOrchestrator` service that calls the three real
  module entry points (as you found them) in sequence, threading `run_id` through every
  call and log line.
- The per-run working directory layout so step outputs chain cleanly (e.g.
  `runs/{run_id}/input/`, `runs/{run_id}/exec/`, plus the converter's own output dir).
- A `FileResolver` abstraction hiding Databricks-Volume-vs-local: production copies the
  Volume file to the local working dir; local mode returns the path unchanged.
- A Mermaid sequence/flow diagram of `POST /v0/start` → split → execute →
  convert+aggregate → response.

### Step 2 — API contract (Pydantic v2)
- `StartConversionRequest`: the SAS file path (string), plus optional fields if useful.
- `StartConversionResponse`: `{"message": "Conversion process started successfully",
  "run_id": "<uuid4>"}`.
- `ErrorResponse`: a consistent error shape; never leak stack traces or internal paths.
- HTTP status codes: 200 success, 400 bad input, 422 validation, 500 pipeline failure
  (with `run_id` echoed for traceability).

### Step 3 — Key design decisions (recommend one for each, citing the code)
- **Versioning**: `APIRouter(prefix="/v0")` + `app.include_router(...)`. No third-party
  versioning package — it cannot be assumed installed offline.
- **Converter integration**: based on what you found in Step 0 — import-and-await an
  internal callable if one exists, otherwise invoke the script via
  `asyncio.create_subprocess_exec` with the exact documented args. State which the real
  code supports.
- **Configuration**: `pydantic-settings` `Settings` with `.env` support — or a plain
  `Settings` class over `os.environ` if `pydantic-settings` is not already a repo
  dependency. Detect Databricks via `os.getenv("DATABRICKS_RUNTIME_VERSION")`.
- **Logging**: structured logging with `run_id` on every line. Prefer stdlib `logging`
  with a `LoggerAdapter`/filter unless the repo already uses structlog/loguru.
- **Error handling**: centralized FastAPI exception handlers + a small custom exception
  hierarchy carrying the failing step name and `run_id`.
- **Dependency injection**: provide `Settings`, `FileResolver`, and
  `ConversionOrchestrator` via FastAPI `Depends`.
- **Execution model**: run the pipeline **synchronously** within the request for v0 so the
  `curl` acceptance test produces files before the response returns. Note background
  execution + status polling as a future extension.

### Step 4 — Non-functional requirements
- **Security**: validate/normalize the input path; reject traversal; never echo internal
  paths. SSH keys and Databricks tokens come from env/secrets, never request bodies.
- **Observability**: `run_id` in every log line and error response; log start/end + timing
  of each module call.
- **Local dev experience**: `LOCAL_MODE=true` skips Databricks download; one documented
  `curl` reproduces the full run locally.
- **Extensibility**: status-polling endpoint, background execution, per-step retry,
  multi-version routers — note where each slots in.

### Step 5 — Folder structure (exact, ready to create)
Provide a concrete tree consistent with the repo's existing layout, e.g.:
```
app/backend/
├── main.py
├── config.py
├── dependencies.py
├── routers/v0/start.py
├── schemas/conversion.py
├── services/conversion_orchestrator.py
├── utils/file_resolver.py
├── core/logging.py
└── core/exceptions.py
tests/
```

### Step 6 — Persist the design for the implementation phase
Once the user approves the design, write it to `.claude/handoff/akit-433-design.md` so the
implementation skill can consume it without you re-explaining. Include: the approved folder
structure, the API contract (the Pydantic models), the resolved key design decisions
(especially the converter integration choice), and the per-run working-directory layout.
This file plus `.claude/handoff/akit-433-research.md` are the complete input to the
implementation phase.

## Hard rules
- **Get the codebase research (handoff file) before designing.** Do not design from the hints alone.
- **Design only — write no implementation code in this phase.**
- Reflect the real 3-module mapping; if the code differs from the hints, follow the code.
- Do not reference external URLs or assume internet/package access.
- End with: **"Design approved? Reply 'IMPLEMENT' and I will switch to the
  fastapi-sas-orchestrator-implementer skill."**

Start with Step 0 (obtain the research handoff file), then deliver Steps 1–5, and finish with Step 6 once the design is approved.
