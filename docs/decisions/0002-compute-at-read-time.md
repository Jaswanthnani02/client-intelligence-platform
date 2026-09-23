# 0002 — Store raw signals, compose scores at read time

**Status:** Accepted, reversing an earlier decision of mine.

## Context

Account managers wanted a single indicator of how a client was doing. The obvious answer, and
the one I built first, was a composite score computed during ingestion and stored alongside the
client's data. Cheap to read, trivial to sort by, easy to put in a list.

It worked. People used it. Then it started being wrong in a way nobody could see.

The score was a weighted combination of ticket volume, resolution times, satisfaction responses
and endpoint health. Every one of those inputs moves continuously. The stored score did not. It
moved only when a recomputation happened to run over that client.

So a client record could show a score computed six weeks earlier against facts that had since
changed materially. Nothing in the interface distinguished a score computed an hour ago from one
computed in the previous quarter. Someone would open a record, see a reassuring letter grade, and
walk into a conversation with a stale conclusion.

Worse, the weights changed as we learned. Every weight change invalidated every stored score, and
recomputing them all was expensive enough that it happened on a schedule rather than immediately.
For a window after each change, stored scores were the output of a formula that no longer existed.

## Decision

**Delete the stored scores.** Persist only raw signals — counts, percentages, timestamps, all
directly measured, all individually meaningful. Compose any composite at the moment it is
requested, from whatever the signals say right then.

Removing the stored scores also meant removing the letter grades, the risk bands, and every
derived value that had been persisted. This was a deletion of working, used code.

## Consequences

**A score can no longer be stale**, because there is no score until you ask for one. The inputs
are current by construction.

**Weight changes take effect immediately and retroactively.** Changing a formula changes every
answer from the next read onward. No recomputation backlog, no window of inconsistency.

**Inputs travel with the output.** Because composition happens at read time, the response can
carry the signals it was built from. A number you can interrogate is treated differently from a
number you cannot, and that changed how people used it.

**Reads cost more.** Composition is arithmetic over a small document, so the cost is small, but
it is not free and it is paid on every request rather than once per ingestion.

**Sorting a whole portfolio by score is harder.** You cannot index a value you do not store.
Cross-client ranking has to compose over a set, which constrains how those queries are written.
This is the real cost of the decision, and it is a permanent one.

**It required removing a feature people liked.** That is an organizational cost, not a technical
one, and it was the expensive part.

## What I would reconsider

**Nothing about the decision. Everything about the timing.** The reasoning above was available
before I wrote the first stored score. I did not think it through; I reached for the obvious
implementation and discovered the flaw by operating it.

The general rule I took from this: **a derived value stored separately from its inputs is a cache
with no invalidation strategy.** Sometimes that is acceptable, with an explicit staleness marker
and a deliberate refresh policy. It is never acceptable by accident, and "compute it once and
store it" is the shape mistakes take.

If portfolio-wide ranking became a primary access pattern, I would revisit this — but as an
explicit materialized view with a visible computation timestamp and a documented refresh
guarantee, never as a value silently living next to the data it summarizes.
