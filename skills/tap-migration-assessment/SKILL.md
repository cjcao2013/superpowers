---
name: tap-migration-assessment
description: Use when assessing a test automation project for TAP migration. Brainstorms the project, scans it via parallel agents, reads TAP's capability manifest, and produces a gap-only requirements document (tap-requirements.md).
---

# TAP Migration Assessment

## Context

**TAP** (Test Automation Platform) is a platform that manages test assets, orchestrates execution, and collects results. Migration to TAP means:
- Test cases and data move from local files → **managed by TAP**
- Execution is triggered by CI/CD scripts → **scheduled by TAP**
- Results move from local logs → **collected by TAP**

The underlying test framework and test logic do not change.

**This skill** runs on an existing test automation project that is preparing to migrate to TAP. It is Step 1 of the migration — the agent's only deep role in the process. Steps 2–5 (gap resolution, data import, validation, cutover) are carried out by humans.

## Core Principle

**Gap = what the project needs − what TAP already supports. You need both sides to compute it.**

Writing requirements without first reading TAP's capability manifest produces a wishlist, not a gap analysis. The TAP team will ignore or re-implement things unnecessarily. Establish both sides first, then document only the delta.

## The Iron Law

```
UNDERSTAND THE PROJECT FIRST.
NEVER ASSUME ITS STRUCTURE.
EVERY PROJECT IS DIFFERENT.
```

---

## Step 1: Broad Scan

Scan the project directly before dispatching any sub-agents. Do NOT ask the user. Do NOT make assumptions.

Read these, in order:

1. **Directory structure** — `ls` the project root and test directories. Note where tests, data, and results live.
2. **README / docs** — Any file at the root that describes the project.
3. **Package manifest** — `requirements.txt`, `package.json`, `pom.xml`, `Pipfile`, `pyproject.toml`, `build.gradle`, or equivalent. Identifies the test framework.
4. **CI/CD entry points** — `.github/workflows/`, `Jenkinsfile`, `Makefile`, `.gitlab-ci.yml`, or equivalent. Note exact file paths, do not read deeply yet.
5. **Test files** — Look at the test directory. What file extension do test files use? Read one sample file to confirm how a test case is defined.

After reading, record:

| Finding | Why It Matters |
|---|---|
| Test framework (name + version) | Shapes what sub-agents look for in Step 2 |
| Test directory path(s) | Tells Agent 1 exactly where to look |
| Test case format (file extension + how a test case is defined) | Tells Agent 1 what patterns to search for |
| Approximate test case count | Scopes the migration effort |
| Test data directory path(s) | Tells Agent 2 where data lives |
| CI/CD config file path(s) | Agent 3 reads these in Step 2 |
| Results/output directory path(s) | Tells Agent 4 where to look for result files |

Do not proceed to Step 2 until all seven findings are recorded.

---

## Step 2: Dispatch Parallel Agents

**REQUIRED: Use `superpowers:dispatching-parallel-agents`.**

Four agents organized around what TAP needs to take over:

```
Agent 1 — Test Cases
  Task: Find and inventory all test cases.
  Answer: How many? Where do they live? How are they structured and named?

Agent 2 — Test Data
  Task: Find all test data sources.
  Answer: Format, location, volume, per-environment split.
  Detect: Sensitive columns (password/token/secret/key in name).

Agent 3 — Execution & CI/CD
  Task: Read all CI/CD config, pipeline scripts, Makefile, run scripts.
  Answer: Exact command to run tests. What variables or parameters are
          injected at runtime? How are environments selected?

Agent 4 — Results & Reporting
  Task: Find what output the test runner produces.
  Answer: Format (XML, JSON, HTML), file location, what a passing vs
          failing run looks like on disk.
```

Synthesize before proceeding:
- What is the actual test case count and where do they live?
- What test data formats and volumes does TAP need to handle?
- What does TAP need to inject at execution time?
- What result format must TAP collect?
- Any sensitive data requiring vault handling?

---

## Step 3: Read TAP Capability Manifest

Look for `tap-capabilities.json` at project root or `~/.tap/capabilities.json`.

**If not found — STOP.** Output this message verbatim:

> `tap-capabilities.json` is required before gap analysis can run.
> Ask the TAP team to fill in the template below, save it as `tap-capabilities.json` at the project root, and re-run this skill.
>
> ```json
> {
>   "version": "0.1",
>   "import": {
>     "_comment": "File formats TAP accepts when importing test cases and test data (e.g. 'csv', 'xlsx', 'json', 'xml'). Leave empty if not yet supported.",
>     "test_cases": [],
>     "test_data": []
>   },
>   "apis": {
>     "_comment": "API endpoint paths TAP exposes that are relevant to migration — for registering test cases, triggering runs, and retrieving results (e.g. '/api/v1/testcases', '/api/v1/runs').",
>     "available": []
>   },
>   "execution": {
>     "_comment": "variable_injection: set to true if TAP can pass runtime variables (e.g. ENV, BASE_URL) into the test runner when triggering a run. supported_flags: flags TAP forwards to the test runner command (e.g. '--env', '--tags').",
>     "variable_injection": false,
>     "supported_flags": []
>   },
>   "results": {
>     "_comment": "Result formats TAP can collect and parse after a run completes (e.g. 'junit-xml', 'json', 'html').",
>     "formats": []
>   }
> }
> ```

Do NOT continue without it. Do NOT assume TAP capabilities.

Note the `version` field — reference it in tap-requirements.md.

---

## Step 4: Gap Analysis

Compare what the project needs (from Step 2 synthesis) against what TAP already supports (from Step 3). Record only the delta.

| Project Need | Where to Check in tap-capabilities.json |
|---|---|
| Test case import format | `import.test_cases` |
| Test data import format | `import.test_data` |
| Specific API endpoints needed | `apis.available` |
| Runtime variable injection | `execution.variable_injection` |
| Result collection format | `results.formats` |

Every missing item is a gap. Every present item is already satisfied — do not list it as a requirement.

---

## Step 5: Write tap-requirements.md

```markdown
# TAP Requirements for [project name]

Generated: [today's date]
TAP capabilities version: [version from tap-capabilities.json]
Test framework: [detected framework]
Test case count: [actual count and source]

## Gaps — TAP Must Implement

### [Gap name]
- What: [specific API, endpoint, or capability]
- Why: [what breaks in the migration without this]
- Acceptance: [how to verify TAP has implemented it]

## Already Satisfied

- [one line per item TAP already covers — no detail needed]
```

Fill every gap with real values from agent synthesis. Generic placeholders are not acceptable.

---

## Red Flags

- You asked the user about the project instead of reading files in Step 1
- You assumed the framework without confirming in Step 1
- You dispatched sub-agents before recording all seven findings from Step 1
- You wrote tap-requirements.md before reading tap-capabilities.json
- You listed capabilities TAP already supports as requirements

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "I'll just ask the user what framework they use" | Files are ground truth. User memory may be wrong or incomplete. |
| "I can see the files, I know the structure" | You don't know until you actually read them in Step 1. |
| "TAP can probably handle this" | Check tap-capabilities.json. Do not guess. |
| "I'll list all requirements to be safe" | That's a wishlist. TAP team will ignore or re-implement unnecessarily. |
