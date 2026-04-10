# Shell Scripting — Deep Prep

> Dedicated folder for acing shell scripting rounds at product companies.
> 12 years of experience → senior level bar: not just "write a loop", but production-grade scripts.

## Folder Structure

```
12-shell-scripting/
├── README.md               ← This file
├── concepts.md             ← Every concept you must know cold
├── exercises.md            ← 60 coding exercises (easy → expert)
└── solutions.md            ← Full solutions with explanation + gotchas
```

## What Senior Interviews Test

| Area | What They Ask |
|---|---|
| Text processing | Parse logs, extract fields, aggregate data with `awk`, `sed`, `grep` |
| Process management | Background jobs, signals, traps, PID tracking |
| Error handling | `set -euo pipefail`, trap EXIT, meaningful error messages |
| File operations | Find, bulk rename, diff, checksums, permissions |
| Concurrency | Parallel execution, job control, wait, race conditions |
| Networking | `curl`, `nc`, port checks, retry logic |
| System info | Disk, memory, CPU, process monitoring |
| String manipulation | Parameter expansion, regex, pattern matching |
| DevOps-specific | K8s health checks, deploy validation, log analysis, alerting |
| Defensive scripting | Input validation, lock files, idempotency |

## How to Use This Folder

1. Read `concepts.md` once end-to-end — build the mental model
2. Open `exercises.md` — attempt each exercise WITHOUT looking at solutions
3. Check `solutions.md` only after you've written your version
4. For each exercise: ask "is this production-safe?" — check error handling, edge cases
5. Re-attempt exercises you struggled with after 2 days

## Interview Tips

- Always say `set -euo pipefail` before writing anything else — it signals seniority immediately
- When given a problem, ask: "Should this be idempotent? Is there concurrent execution?" before coding
- Name variables in UPPER_CASE for env vars / globals, lower_case for locals
- Every script should have a usage message if required args are missing
- Prefer `[[ ]]` over `[ ]` — it's safer and supports regex
