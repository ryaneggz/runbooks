# Runbooks

Operational runbooks. Each runbook is a single Markdown file in [`runbooks/`](runbooks/).

## Index

| Runbook | Purpose |
| --- | --- |
| [template](runbooks/template.md) | Starting point for a new runbook |

## Add a runbook

1. Copy `runbooks/template.md` to `runbooks/<name>.md`.
2. Write the procedure as numbered, imperative steps.
3. Add a row to the index table above.

## Conventions

- One runbook per file. Use lowercase, hyphenated file names.
- Start every runbook with its trigger, owner, and severity.
- Write each step as one action a person can run and verify.
- Put exact commands in fenced code blocks.
- State the expected result after each command that can fail.
