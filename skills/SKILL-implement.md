---
name: fastapi-sas-orchestrator-implementer
description: Implement (build the actual code for) the production-ready FastAPI v0 backend that orchestrates the SAS-to-PySpark conversion pipeline for Jira ticket AKIT-433. Use this skill whenever the task is to build, code, or wire up the POST /v0/start endpoint, the ConversionOrchestrator service over the sas_splitter / sas_executor / sas_converter modules, file handling, config, or the curl acceptance test — even if the user just says "implement the conversion API" or "build the orchestrator". Pairs with the fastapi-sas-orchestrator-designer skill, which handles the design phase that precedes this one.
---

# FastAPI SAS-to-PySpark Orchestrator — Implementation Phase (Staff-Level)

You are the staff engineer building the AKIT-433 backend. Build the backend the design
phase approved. Keep it a **small, clean FastAPI backend** — do not over-engineer.

> This skill is consumed by an agent with **full access to the
> existing repository**. The repository is your single source of truth. The "Module Hints"
> section below is only a rough orientation from external screenshots — it is NOT
> authoritative and may be stale or wrong. You must read the actual module code and
> implement against what you find there, never against the hints.

## The Ticket (AKIT-433)

- Single entry point: `POST /v0/start`.
- Input: path to a SAS file in a **Databricks Volume** (production) OR a **local path**
  (local testing, download skipped — controlled by `LOCAL_MODE`).
- Orchestrate the pipeline **sequentially** so output files are **identical** to running
  the steps manually.
- Generate a `run_id` (UUID4); return it in the success response.
- Acceptance: a `curl` call triggers the full pipeline and produces identical output files.

## Step 1 (NON-NEGOTIABLE): Load the research and design handoff files

The codebase was already researched by the design/research phase. Do not re-research it
from scratch — consume the handoff:

1. **Read `.claude/handoff/akit-433-research.md`** in full. This is your trusted research
   result: the real module signatures, return types, manual invocation, on-disk data
   flow, the converter integration verdict, the dependency inventory, and repo
   structure/style. Treat it as ground truth.
2. **Read `.claude/handoff/akit-433-design.md`** in full. This is the approved design:
   folder structure, API contract, resolved design decisions, working-dir layout. Build
   exactly this — do not redesign.
3. **If either file is missing or incomplete**, fall back: delegate to the
   `sas-codebase-researcher` subagent to (re)generate the research file, or do the
   research inline; and ask the user for the approved design or run the design skill
   first. Never implement against the "Module Hints" alone.
4. **Spot-check, do not re-derive.** While implementing, if a handoff statement clearly
   contradicts the actual code (e.g. a signature changed), the code wins: follow the code,
   fix the line in the research file, and flag the discrepancy to the user. Otherwise
   trust the handoff so you are not paying the research cost twice.

**Report** which handoff files you loaded and the key facts you are building on (converter
integration verdict, pydantic version, module entry points) before writing code.

## Critical: 4 Ticket Steps → 3 Module Calls

The ticket says "4 steps" (split, execute, convert, aggregate) but the codebase has
**three modules** — the converter combines *convert + aggregate* in one run. Confirm this
in the real code, then make **three** orchestrator calls, in order: split → execute →
convert+aggregate (full run, not a convert-only mode). Do not write a fourth call.

## Module Hints (orientation only — verify against the real source)

Rough notes from external screenshots. Use them only to know where to start reading.

- **`sas_splitter`** — likely under `ai_orchestrator.sas_splitter.splitter`; a single-file
  split appears to write `*_split.sas` + `*_metadata.json` and inject Snowflake checkpoint
  macros. Confirm the real entry point, signature, and output filenames.
- **`sas_executor`** — likely under `ai_orchestrator.sas_executor.file_executor`; appears
  to run a `.sas` file on a remote SAS server over SSH and return a result object with
  success/log/exit/error fields. Executing the checkpoint file appears to populate
  Snowflake checkpoint tables the converter reconciles against — so execute must precede
  convert. Confirm this.
- **`sas_converter`** — appears driven by `run_batch_and_aggregate.py`, converting all
  chunks and aggregating in one run from a folder of splitter outputs. Confirm the input
  files it requires and whether it is importable or argparse-only.

## Converter integration — use the verdict from the research handoff

- **Option A (preferred, if a callable exists):** import the converter's
  function/coroutine and call/`await` it inside the orchestrator.
- **Option B (fallback, if argparse-only):** invoke the script with
  `asyncio.create_subprocess_exec`, passing the exact arguments the manual workflow uses.
  Capture stdout/stderr into the run log; non-zero exit → raise a pipeline error.

Either way, do not modify the converter's behavior — output files must stay byte-identical
to a manual run.

## Implementation order

1. **Config** (`config.py`): `Settings` with `LOCAL_MODE`, `INPUT_FILE_PATH` /
   `FILE_BASE_PATH`, SSH params, Databricks Volume settings, converter knobs. Detect
   Databricks via `os.getenv("DATABRICKS_RUNTIME_VERSION")`. Use `pydantic-settings` only
   if the repo already has it; otherwise a plain class over `os.environ`.
2. **File resolver** (`utils/file_resolver.py`): production → copy the Volume file into the
   run's working dir; local → return the path unchanged. Selected by `LOCAL_MODE`.
3. **Logging** (`core/logging.py`): structured logging with `run_id` on every line via a
   `LoggerAdapter`/filter. Stdlib `logging` unless the repo already uses structlog/loguru.
4. **Exceptions** (`core/exceptions.py`): a small hierarchy (e.g. `PipelineStepError` with
   the failing step name + `run_id`) and FastAPI exception handlers returning the
   `ErrorResponse` shape without leaking internals.
5. **Orchestrator** (`services/conversion_orchestrator.py`): generate `run_id` (UUID4),
   create the per-run working dirs, then call the three real module entry points in order,
   threading `run_id` and chaining paths between steps. Log start/end + timing per step.
6. **Schemas** (`schemas/conversion.py`): Pydantic v2 `StartConversionRequest`,
   `StartConversionResponse`, `ErrorResponse`.
7. **Router** (`routers/v0/start.py`): `APIRouter(prefix="/v0")`, `POST /start`,
   dependencies via `Depends`. Run the pipeline synchronously so files exist before the
   response returns.
8. **App wiring** (`main.py`): `app.include_router(...)`, register exception handlers,
   ensure `/docs` renders cleanly.

## API contract
- Success response: `{"message": "Conversion process started successfully",
  "run_id": "<uuid4>"}`.
- Versioning: `APIRouter(prefix="/v0")` + `app.include_router(...)`. No third-party
  versioning package.
- Pydantic v2 everywhere; type hints everywhere; follow PEP 8 and the existing repo style.

## Validation — satisfy the acceptance criteria
1. Provide the exact `curl` command that triggers the full pipeline against a local
   `.sas` file (`LOCAL_MODE=true`), e.g.:
   ```
   curl -X POST http://localhost:8000/v0/start \
     -H "Content-Type: application/json" \
     -d '{"sas_file_path": "/path/to/Program.sas"}'
   ```
2. Show that the output files match a manual run of split → execute → convert+aggregate.
3. Add at least a minimal test (or documented manual verification) covering the happy path
   and one failure path (e.g. missing input file → 400/500 with `run_id`).

## Hard rules
- **Load the research + design handoff files first** — do not re-research or redesign.
- The repository overrides everything; if a handoff file disagrees with the code, follow the code and fix the handoff file.
- **Never over-engineer** — keep it a small FastAPI backend.
- Three module calls, not four. Full convert+aggregate.
- Do not reference external URLs or assume internet/package installs.
- Make local development effortless (`LOCAL_MODE=true` skips all Databricks logic).

When done, say: **"Implementation complete. Ready for review or testing."**
