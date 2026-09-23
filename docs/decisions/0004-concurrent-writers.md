# 0004 — Guarded read-merge-write for a document with two owners

**Status:** Accepted.

## Context

One aggregate document per client is what everything downstream reads. It carries service-desk
signals, satisfaction signals and endpoint-fleet signals in one place, because the alternative —
assembling it per request from several sources — was the latency problem the platform existed to
solve.

Two processes maintain it. One recomputes the service-desk and satisfaction portion whenever
tickets or surveys are ingested. The other writes the endpoint-fleet portion on its own,
different schedule. Neither knows the other's fields. Both run on timers that will eventually
overlap.

The document store's natural write operation replaces the whole document. That produces this:

1. Process A reads the document, computes its fields, holds the result.
2. Process B reads the same document, computes its fields, writes.
3. Process A writes the result it computed in step 1.

B's fields are gone. Not corrupted — **absent**, replaced by the values A read before B wrote.

There is no error. The write succeeds. The fields simply stop updating, and because they are a
minority of a large document, nobody notices until someone asks why endpoint data looks frozen.
That is a question people ask weeks late, if at all.

## Decision

Every write to that document goes through a **single shared function** implementing an
etag-guarded read-merge-write:

Read the current document with its version token. Merge the caller's fields over it, preserving
everything not supplied. Write with the version token as a precondition. If the store rejects the
write because the version moved, re-read and retry.

Optimistic concurrency. No distributed lock, no coordination between the two schedules, no
ordering requirement.

The important part is not the algorithm, which is standard. It is that the function is the
**only** sanctioned way to touch that document, and that this is stated where someone about to
write the wrong thing will read it.

## Consequences

**Concurrent writers are safe** without either knowing the other exists. A third writer could be
added tomorrow with no changes to the existing two.

**No locking infrastructure.** Nothing to deploy, nothing to expire, no deadlock, no lease.
Under this contention — two writers, seconds of overlap, minutes apart — collisions are rare and
a retry resolves them.

**The guarantee is bypassable.** Any code holding a client to the store can call the raw
replace operation. The protection is discipline plus review, not a mechanism. This is the
weakness of the design and I want to be honest about it rather than present it as airtight.

**Writes cost an extra read.** Irrelevant at this volume, and it would matter at a much higher
one.

## What I would reconsider

**I would make bypass harder rather than merely discouraged.** Options I did not take: a
dedicated wrapper type for that document that exposes only the merging write; a test that scans
for raw write calls against that collection and fails if it finds one outside the sanctioned
function. The second is cheap and mechanical, and I should have written it — it is precisely the
"make the invariant executable" lesson from elsewhere in this system, which I applied in other
places and not here.

**The retry behaviour deserved more thought than it got.** It retries a small number of times and
then fails the pass. Under sustained contention that becomes a silent gap in recomputation rather
than a loud failure. A metric on retry exhaustion would have been a better early warning than the
one we actually had, which was a person noticing.

**More broadly:** a shared mutable structure with more than one owner is a design smell worth
challenging before solving. I considered splitting the document so each writer owned its own, and
rejected it because the read path would then need to assemble several documents — reintroducing
exactly the latency the aggregate exists to avoid. That remains the right call for this workload,
but it was a genuine tradeoff, not an obvious one.
