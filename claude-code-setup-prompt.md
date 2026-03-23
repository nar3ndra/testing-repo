# Prompt to Paste into Claude Code

> Copy everything below the line and paste it into your Claude Code session.

---

I need you to scaffold a complete `.claude/` folder structure for my project. This is a **SAS-to-PySpark code converter** that uses the Claude Agent SDK with Claude Opus 4.5 models. The project reads legacy SAS programs, converts them to PySpark, and executes them on Databricks. I also use Snowflake as a downstream data store.

## Project Context

- **Stack**: Python, Claude Agent SDK, Claude Opus 4.5, PySpark, Databricks, Snowflake
- **What it does**: Takes SAS `.sas` files as input, parses them, converts to equivalent PySpark code, validates the output, and optionally executes on a Databricks cluster
- **Key patterns**: Agentic AI orchestration, SAS parsing, PySpark code generation, Databricks execution layer, Snowflake writes
- **Config approach**: `.env` files for secrets (API keys, Databricks tokens, Snowflake creds), centralized config module
- **Naming conventions**: Timestamped run names, SHA-256 hash-based folder organization for converted outputs

## What I Need You to Create

Generate the full `.claude/` folder structure following this anatomy:

```
your-project/
├── CLAUDE.md                    ← team instructions, committed
├── CLAUDE.local.md              ← personal overrides, gitignored
├── .claude/
│   ├── settings.json            ← permissions + config, committed
│   ├── settings.local.json      ← personal permissions, gitignored
│   ├── commands/                ← custom slash commands
│   │   ├── convert.md           → /project:convert
│   │   ├── validate.md          → /project:validate
│   │   ├── execute.md           → /project:execute
│   │   └── review.md            → /project:review
│   ├── rules/                   ← modular instruction files
│   │   ├── code-style.md
│   │   ├── sas-conversion.md
│   │   ├── pyspark-patterns.md
│   │   ├── testing.md
│   │   └── api-conventions.md
│   ├── skills/                  ← auto-invoked workflows
│   │   ├── sas-parser/
│   │   │   └── SKILL.md
│   │   ├── pyspark-generator/
│   │   │   └── SKILL.md
│   │   ├── databricks-executor/
│   │   │   └── SKILL.md
│   │   └── snowflake-writer/
│   │   │   └── SKILL.md
│   └── agents/                  ← subagent personas
│       ├── sas-analyst.md
│       ├── pyspark-reviewer.md
│       └── test-generator.md
```

## Detailed Requirements for Each File

### CLAUDE.md (Root)
Write a comprehensive project instruction file that includes:
- Project overview (SAS-to-PySpark converter using Claude Agent SDK)
- Architecture description (agent orchestration flow: parse SAS → plan conversion → generate PySpark → validate → execute)
- Tech stack with versions where relevant
- Key directories and their purpose
- Coding conventions: Python type hints, docstrings, error handling patterns
- Environment setup: `.env` file structure, required env vars (ANTHROPIC_API_KEY, DATABRICKS_HOST, DATABRICKS_TOKEN, SNOWFLAKE_ACCOUNT, etc.)
- Build/run/test commands
- Git workflow and branch naming
- IMPORTANT: Include a section on SAS-to-PySpark conversion rules (common SAS constructs and their PySpark equivalents like DATA steps → DataFrame ops, PROC SQL → spark.sql, PROC SORT → orderBy, macros → Python functions, SAS formats → PySpark UDFs)
- Include a section on known gotchas (SAS missing values vs PySpark nulls, SAS date handling, implicit type coercion, SAS merge vs PySpark join semantics, RETAIN equivalent in PySpark)

### CLAUDE.local.md
Personal preferences template with placeholders for:
- Preferred editor/IDE
- Local Databricks workspace URL
- Personal coding style preferences

### .claude/settings.json
Configure permissions:
- **allow**: Read, Grep, LS, Bash(python *), Bash(pytest *), Bash(pip *), Bash(databricks *), Bash(git *)
- **deny**: Read(./.env), Read(./.env.*), Read(./secrets/**), Read(./config/credentials.json), Bash(rm -rf *), WebFetch
- Include any relevant model or attribution settings

### .claude/commands/

**convert.md**: Slash command for `/project:convert` — Takes a SAS file path, runs the full conversion pipeline (parse → plan → generate → validate), outputs the PySpark equivalent

**validate.md**: Slash command for `/project:validate` — Takes a generated PySpark file, runs static analysis, checks for common conversion errors, runs test cases if available

**execute.md**: Slash command for `/project:execute` — Submits a validated PySpark script to Databricks, monitors the job, reports results

**review.md**: Slash command for `/project:review` — Code review focused on SAS-to-PySpark conversion accuracy, checks for semantic equivalence

### .claude/rules/

**code-style.md**: Python coding standards — PEP 8, type hints required, Google-style docstrings, structured logging, error handling with custom exceptions

**sas-conversion.md**: Rules for how SAS constructs map to PySpark — this is the most critical rule file. Include mappings for DATA steps, PROC SQL, PROC SORT, PROC MEANS/SUMMARY, PROC FREQ, SAS macros, SAS formats, RETAIN, BY-group processing, OUTPUT statements, MERGE, FIRST./LAST. processing, arrays, etc.

**pyspark-patterns.md**: PySpark best practices — prefer DataFrame API over SQL strings, use column expressions, broadcast small tables, partition strategy, avoid collect() on large data, handle nulls explicitly, window functions for SAS-like operations

**testing.md**: Testing approach — unit tests for individual SAS construct conversions, integration tests comparing SAS output vs PySpark output row-by-row, pytest fixtures for SparkSession management

**api-conventions.md**: Claude Agent SDK patterns — how agents are structured, tool definitions, message passing, error handling in agent loops

### .claude/skills/

**sas-parser/SKILL.md**: Auto-triggers when working with .sas files. Describes how to parse SAS code, identify constructs (DATA steps, PROCs, macros, includes), build an AST or structured representation.

**pyspark-generator/SKILL.md**: Auto-triggers when generating .py files from SAS conversions. Describes the code generation approach, template patterns, import management, SparkSession handling.

**databricks-executor/SKILL.md**: Auto-triggers when executing or deploying to Databricks. Describes the Databricks REST API or CLI usage, job submission, cluster management, notebook export format.

**snowflake-writer/SKILL.md**: Auto-triggers when writing data to Snowflake. Describes the Snowflake connector patterns, Spark-Snowflake connector vs direct write, schema mapping from SAS/PySpark types to Snowflake types.

### .claude/agents/

**sas-analyst.md**: Subagent specialized in understanding SAS code — reads SAS programs, identifies business logic, flags complex constructs that need human review, documents data lineage.

**pyspark-reviewer.md**: Subagent specialized in reviewing generated PySpark code — checks for performance issues, Spark anti-patterns, null handling correctness, partition strategy, and semantic equivalence with original SAS logic.

**test-generator.md**: Subagent that generates test cases — creates pytest test files that compare SAS output (if available) with PySpark output, generates edge case tests for null handling, date conversions, type coercion.

## Additional Instructions

1. Make every file production-quality and detailed — not placeholder stubs
2. The CLAUDE.md should be thorough enough that a new team member reading it understands the entire project
3. Rules files should contain actionable, specific instructions — not generic advice
4. Skills should have proper YAML frontmatter with name, description, and trigger conditions
5. Agents should have YAML frontmatter with model preferences and tool permissions
6. Add a `.gitignore` entry list at the bottom of CLAUDE.md noting which files should be gitignored (CLAUDE.local.md, settings.local.json)
7. Commit the `.claude/` folder to git (except local files)

Please create all these files now. Start with CLAUDE.md, then .claude/settings.json, then work through commands, rules, skills, and agents in order.
