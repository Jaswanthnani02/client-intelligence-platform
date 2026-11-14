# Decision records

Five decisions that shaped the rest of the system. Each states the context, what was decided,
what it cost, and what I would reconsider.

| # | Decision | The thing it prevents |
|---|---|---|
| [0001](0001-single-writer-boundary.md) | One service writes; the other is strictly read-only | Ambiguity about which component produced a wrong value |
| [0002](0002-compute-at-read-time.md) | Store raw signals, compose scores when asked | Derived values silently rotting as their inputs move |
| [0003](0003-lagging-watermarks.md) | Track ingestion progress behind the true maximum | Records updated mid-pass being skipped forever |
| [0004](0004-concurrent-writers.md) | Guarded read-merge-write for a shared document | One process silently erasing another's fields |
| [0005](0005-per-call-authorization.md) | Authorize every call on role and tenant | A permission mistake defaulting to over-exposure |

## Format

Each record follows the same shape: **Context** (what was true that forced a choice),
**Decision**, **Consequences** (including the costs, not only the benefits), and **What I would
reconsider**.

The last section is the one worth reading. A decision record that only justifies itself is
marketing. Every one of these has a cost I would weigh differently with what I know now, and
one of them describes reversing an earlier decision of my own.
