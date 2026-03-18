---
name: experiment
description: Pick the most promising untested idea from TODO.md, implement it, bench-test it, create a clean branch from master, and prepare Fishtest submission details.
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
argument-hint: [item-number]
---

# Stockfish Experiment Workflow

Run a complete experiment cycle: select idea, implement, test, branch, and prepare for Fishtest.

## Step 1: Select the idea

- If `$ARGUMENTS` specifies an item number, use that item from TODO.md
- Otherwise, scan TODO.md for the first `### ` header ending in `[READY]`
- Extract from the selected item: ID, title, tier, file(s), branch name, rationale, and diff block(s)
- Diff blocks use `~~~diff` fencing. Lines starting with `- ` give `old_string`; lines starting with `+ ` give `new_string` for the Edit tool (strip the leading `- `/`+ ` prefix to get exact text)
- If a diff block contains a `File:` annotation, apply that edit to the specified file; otherwise use the file from the metadata line
- Items with multiple diff blocks require multiple Edit tool calls
- Print the selected item's ID, title, and rationale

## Step 2: Understand the code

- Read the exact source lines referenced in the TODO item
- Read surrounding context (20 lines before and after) to understand the formula
- Identify all three things: what the current code does, what the change is, and why it should help

## Step 3: Implement the change

- Stash any uncommitted work on the current branch: `git stash`
- Create a clean branch from master: `git checkout master && git checkout -b experiment/<descriptive-name>`
- Make the minimal code change described in the TODO item
- Only modify source files (no markdown, no docs)

## Step 4: Build and bench

- Build: `cd src && make -j build ARCH=native`
- Run bench: `./stockfish bench 2>&1 | tail -5`
- Record the bench node count
- Compare to master bench (2,288,704 at current master) — the number MUST differ for functional changes
- If bench is identical to master, the change has no effect at bench depth. Either:
  - Make the change more aggressive, OR
  - Verify the code path is actually reached at bench depth
- If bench nodes increase by >20%, the change may be too aggressive — consider moderating
- If bench nodes decrease by >20%, verify this is intentional (more pruning) not a bug

## Step 5: Validate

- Run perft to verify no move generation corruption:
  ```
  echo -e "position startpos\ngo perft 5\nquit" | ./stockfish | grep "Nodes searched"
  ```
  Must output: `Nodes searched: 4865609`

## Step 6: Commit and push

- Verify only source files are staged (no .md files): `git status`
- `git add src/<changed-files>`
- Commit with message format:
  ```
  <Short description of change>

  <1-2 sentence explanation of why this should help>

  Bench: <node count>
  ```
- Push: `git push origin experiment/<branch-name>`

## Step 7: Output Fishtest submission details

Fishtest auto-populates **Test Signature** (parses `Bench: <number>` from the last commit)
and **Info** (from the full commit message). The commit message format in Step 6 is critical
for these defaults to work — `Bench: <number>` MUST be the last line.

Print only the fields the user needs to manually fill in on https://tests.stockfishchess.org/tests/run :

```
Test Branch:      experiment/<name>
Base Signature:   2288704
```

Then remind the user to verify that Test Signature and Info auto-populated correctly from the commit.

All other form fields keep their defaults (Test type: STC, Stop rule: SPRT,
SPRT Bounds: Standard STC <0.00,2.00>, Base Branch: master, Threads: 1,
TC: 10+0.1, Hash=16, Auto-purge checked).

## Step 8: Update TODO.md (on a different branch)

- Switch back to the working branch where TODO.md lives
- In TODO.md, replace `[READY]` with `[SUBMITTED]` in the item's `### ` header line
- If the metadata line doesn't already have a `**Branch:**` field, add `| **Branch:** experiment/<name>` to it
- Do NOT commit this change (user will handle it)

## Important constraints

- NEVER include markdown files in the experiment branch commit
- NEVER modify more than one formula per experiment (clean attribution)
- ALWAYS verify bench differs from master before committing
- If the bench takes longer than 15 seconds, something is wrong (competing processes or too-aggressive change)
- The experiment branch must be a single commit on top of master
- Use `git push origin <branch>` to push (remote is `origin` = `tcberkley/Stockfish`)
