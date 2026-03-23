# Prompt for Claude Code — Generate My .claude/ Dev Setup

> Copy everything below the line and paste it into Claude Code at your project root.

---

I need you to scaffold a `.claude/` folder setup for my **development workflow**. This is NOT about creating the project's business logic — I already have that built. I need Claude Code configured so it understands my codebase and helps me develop, debug, test, and iterate efficiently.

## About My Project

This is a **SAS-to-PySpark code converter** built with the Claude Agent SDK and Claude Opus 4.5 models. The repo already contains:
- Multiple agents (converter, aggregator, reconciliation, etc.) each with their own skill files
- A skills folder with SKILL.md files for each agent
- Python codebase using PySpark, Databricks, Snowflake
- `.env` files for configuration (API keys, Databricks tokens, Snowflake creds)
- SHA-256 hash-based folder organization for converted outputs
- Timestamped run naming conventions

**Before generating anything, run the following:**
1. `find . -type f -name "*.py" | head -40` — to see my Python file structure
2. `find . -type f -name "*.md" | head -20` — to see existing markdown/skill files
3. `find . -name "SKILL.md"` — to see my existing skills
4. `cat .env.example || cat .env.sample || echo "no env example found"` — to see env var patterns
5. `ls -la` — to see root directory structure
6. `cat requirements.txt || cat pyproject.toml || echo "no deps file found"` — to see dependencies

**Use what you discover from those commands to fill in the actual details below — don't use placeholders.**

## What to Generate

```
CLAUDE.md                        ← project instructions for Claude Code
CLAUDE.local.md                  ← my personal overrides (gitignored)
.claude/
├── settings.json                ← permissions + config, committed
├── settings.local.json          ← personal permissions, gitignored
├── commands/
│   ├── dev.md                   → /project:dev  (spin up dev environment)
│   ├── test.md                  → /project:test (run tests)
│   ├── debug.md                 → /project:debug (debug a failing conversion)
│   ├── pr.md                    → /project:pr (prepare a PR)
│   └── refactor.md              → /project:refactor (refactor a module)
├── rules/
│   ├── code-style.md            ← Python conventions for this project
│   ├── testing.md               ← how we write and run tests
│   ├── agent-sdk.md             ← Claude Agent SDK patterns we follow
│   ├── git-workflow.md          ← branching, commits, PRs
│   └── security.md              ← secrets handling, .env rules
└── agents/
    ├── code-reviewer.md         ← reviews my code changes
    └── test-writer.md           ← generates tests for my code
```

## Requirements for Each File

### CLAUDE.md
Scan my actual repo structure and write this file based on what you find. Include:
- **Project overview**: What this project does (derive from the codebase, not generic)
- **Architecture**: Describe the actual folder structure and how components connect — based on what you see in the repo
- **Tech stack**: List actual dependencies from requirements.txt/pyproject.toml
- **Key directories**: Map out the real directories and what lives in each
- **How to run**: Actual commands to run the project, run tests, lint, etc. — derive from Makefile, scripts/, or pyproject.toml if they exist
- **How to test**: Actual test commands and test file locations
- **Environment setup**: List the actual env vars from .env.example or from scanning the code for `os.getenv()` / `os.environ` calls
- **Coding conventions**: Infer from the existing code — check if the project uses type hints, which docstring style, logging approach, error handling patterns
- **Agent architecture**: Describe the actual agents and skills that exist in the repo, how they're orchestrated
- **Git rules**: Branch naming, commit message format (check git log for patterns)
- **What NOT to do**: Add a section with common mistakes specific to this codebase

### CLAUDE.local.md
```markdown
# Personal Dev Preferences
- I prefer concise code with minimal comments — the code should be self-documenting
- When debugging, show me the full stack trace first before suggesting fixes
- I use Databricks workspace at: [fill in your URL]
- Default to running single tests, not the full suite
```

### .claude/settings.json
```json
{
  "permissions": {
    "allow": [
      "Read",
      "Grep",
      "LS",
      "Bash(python *)",
      "Bash(python3 *)",
      "Bash(pytest *)",
      "Bash(pip *)",
      "Bash(git *)",
      "Bash(find *)",
      "Bash(cat *)",
      "Bash(head *)",
      "Bash(tail *)",
      "Bash(wc *)",
      "Bash(grep *)",
      "Bash(make *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials*)",
      "Bash(rm -rf *)"
    ]
  }
}
```

### .claude/commands/

**dev.md** — `/project:dev`: Steps to set up and verify the dev environment. Check Python version, verify dependencies are installed, confirm env vars are set (without reading values), run a quick smoke test.

**test.md** — `/project:test`: Run tests. Accept an optional argument for a specific test file or test function. Default to running the full suite. Show coverage if pytest-cov is installed. Use the actual test directory from the repo.

**debug.md** — `/project:debug`: Debug a failing agent run or conversion. Accept a run ID or file path. Check logs, trace the agent execution flow, identify where it failed, suggest a fix.

**pr.md** — `/project:pr`: Prepare a pull request. Run linting, run tests, generate a diff summary, write a PR description based on the changes, check for any .env or secrets accidentally staged.

**refactor.md** — `/project:refactor`: Refactor a specified module. Read the module, identify code smells (duplication, long functions, poor naming, missing types), propose changes, apply them, run tests to confirm nothing broke.

### .claude/rules/

**code-style.md**: Scan my existing code and infer the actual style being used. Document it so Claude Code follows the same patterns. Include: import ordering, type hint usage, docstring format, logging approach, exception handling, naming conventions.

**testing.md**: Document how tests are structured in this repo. Where test files live, naming conventions, fixtures in use, how to run a single test, how to run the full suite, any test data patterns.

**agent-sdk.md**: Document the Claude Agent SDK patterns used in this project. How agents are defined, how tools are registered, how conversations are managed, how errors are handled in agent loops. Derive this from the actual agent code in the repo.

**git-workflow.md**: Check `git log --oneline -20` and infer the commit message style and branching pattern. Document it.

**security.md**: Rules about secrets handling — never read .env files, never log API keys, never commit credentials, how to reference config values safely.

### .claude/agents/

**code-reviewer.md**: A subagent that reviews code changes with focus on:
- Python best practices and the project's specific conventions
- PySpark performance (avoid collect(), use broadcast for small tables, partition awareness)
- Agent SDK correctness (proper tool definitions, error handling)
- Security (no leaked secrets, safe env var access)

**test-writer.md**: A subagent that generates tests for new or modified code:
- Follows the project's existing test patterns
- Creates pytest tests with appropriate fixtures
- Covers happy path, edge cases, and error cases
- For conversion-related code, generates comparison tests (expected output vs actual)

## Final Steps After Generation

After creating all files:
1. Add to `.gitignore`: `CLAUDE.local.md` and `.claude/settings.local.json`
2. Show me a summary of what was created
3. Run `/init` to cross-reference with the codebase and suggest any refinements to CLAUDE.md
