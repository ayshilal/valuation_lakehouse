# Declarative Automation Bundles Project

This project uses Declarative Automation Bundles (DABs) for deployment. Add project-specific instructions below.

## For AI Agents: Use Databricks AI Tools

**BEFORE any other action, read the `databricks-core` skill.**

It sets you up to work with this project reliably: CLI authentication, profile
selection, data discovery, and the bundle deployment workflow. Without it,
results are often slower and less accurate.

If this skill is not available (Databricks AI Tools are not installed), you can install them for your coding agent in seconds:

```bash
databricks aitools install
```

If the CLI is not installed, see: https://docs.databricks.com/dev-tools/cli/install

---

## Project Instructions

### Read and maintain PROJECT_OVERVIEW.md

`PROJECT_OVERVIEW.md` is this project's living architecture and state document: what the
bundle contains, how dev/prod targets differ, how to test, and the known gotchas.

**At the start of a task:** read it. It will tell you what is real code and what is still
untouched template scaffold, which is not obvious from file names alone.

**Before you finish a task:** update it if your change made any part of it stale. This is part
of the task, not a follow-up — do not leave it for the user to ask. Update it when you:

- add, remove, or rename a transformation, job, pipeline, or other bundle resource
- change targets, variables, catalogs, schemas, or deployment configuration
- add or drop a dependency, or change the Python/tooling setup
- fix something in its "Known issues and gotchas" table (remove the row) or discover a new
  one (add a row)
- replace template sample code with real logic

Bump the **Last updated** date whenever you edit it. Keep it describing what the repo *is*
now — it is not a changelog, and it is not a duplicate of `README.md` (which covers setup and
commands).

<!-- Add further project-specific instructions, coding conventions, or notes below -->
