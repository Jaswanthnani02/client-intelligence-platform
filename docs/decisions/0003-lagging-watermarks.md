# 0003 — Track ingestion progress behind the true maximum

**Status:** Accepted.

## Context

Each source is polled incrementally. A pass asks for everything modified since the last run,
writes what it gets, and records how far it reached. The recorded position is the watermark.

The obvious implementation stores the highest modification timestamp seen in the batch. It is
exact, it is what the data says, and it loses records.

The loss happens like this. Pagination over a set ordered by modification time is not a snapshot.
While a pass is reading page four, the source system updates a record that was on page two. That
record's timestamp moves to *now*, which is ahead of where the pass will finish. The pass
completes, stores the maximum it saw, and the updated record sits behind the new watermark
forever. It is never requested again.

Nothing detects this. There is no error, no gap in a sequence, no count that fails to reconcile.
The record is simply older in your store than in the source, indefinitely.

This is not rare. It is guaranteed for any source with concurrent write traffic and a pass that
takes longer than the interval between updates, which describes every ticketing system during
business hours.

## Decision

Store the watermark **deliberately behind** the true maximum, by a margin derived from how often
the job runs.

Consecutive passes overlap and re-read a window of records they have already seen. Correctness
comes from every write being idempotent — the same record written twice produces the same result
— rather than from the watermark being precise.

The lag margin is derived from the schedule rather than hardcoded, so the two cannot drift apart
silently. A test asserts they match.

## Consequences

**The silent-loss class of bug is eliminated**, not mitigated. A record updated mid-pass is
re-read on the next run.

**Passes do redundant work.** Every run re-processes a window of records that have not changed.
For this workload the overlap was a small fraction of each pass, and the store charges by
operation, so the cost was real but minor.

**Idempotency becomes load-bearing.** Every write path must be safe to repeat. That is a
constraint on all future ingestion code, and it needs to be understood by whoever writes the
next source integration, because violating it turns a safety margin into duplicate records.

**Recovery is easy.** Moving a watermark backwards re-ingests a window. Because writes are
idempotent, that is a safe operation rather than a dangerous one, which made incident response
considerably calmer.

## What I would reconsider

**The derivation caught a real incident, and only barely.** The lag margin is computed from the
job's schedule. Someone changed the schedule without changing the constant, the margin became too
small for the new interval, and a downstream recomputation stopped firing correctly for two weeks
before anyone noticed.

The fix was a test asserting the two agree. The lesson is that **a value derived from another
value needs the derivation to be executable, not documented.** A comment saying "keep this in
sync with the schedule" is a comment that will eventually be wrong.

I would also add a reconciliation job from the beginning rather than after the fact. The lagging
watermark prevents mid-pass loss; it does nothing about a pass that fails entirely, or a source
that changes its ordering semantics. A periodic count comparison between source and store is
cheap and catches the classes this does not.
