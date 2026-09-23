# 0001 — One service writes, the other is strictly read-only

**Status:** Accepted, and held under pressure several times.

## Context

The platform is two independently deployed services sharing one document store.

One ingests from four external systems on a schedule and maintains the derived aggregates. The
other exposes the data to a language model as a set of callable tools, so an account manager can
ask questions conversationally.

The natural design lets both write. The query service already holds a database connection; when
a user asks it to create a ticket, writing directly is one line. Routing that through the other
service is an HTTP call, a serialization boundary, a new failure mode and a latency cost, to
achieve something no user can perceive.

Two things made me reject that.

The query layer is the fastest-changing part of the system and the part handling untrusted input
— ticket text, documentation excerpts, model output. It is where a defect is most likely.

And when data is wrong, the first question is always *what wrote this*. With two writers, that
question requires reading both services' histories and reconciling their schedules.

## Decision

The ingestion service is the **sole writer**. The query service holds a read-only path to the
store. Its one write capability, creating a ticket in the source system, posts to an HTTP route
on the ingestion service rather than writing anything itself.

The boundary is structural, not conventional. There is no write-capable client in the query
service to reach for.

## Consequences

**A wrong value has exactly one suspect.** This paid for itself repeatedly. Every data
investigation started in the same place, and none ever needed "which service did this."

**The blast radius of the riskiest component is bounded.** A defect in the conversational layer
degrades reads. It cannot corrupt the record.

**The write path is slower and can fail in a new way.** The HTTP call has a timeout. When it
fires after the downstream action succeeded, a retry duplicates the work. This is a real cost I
did not fully price in — see below.

**Some features are more awkward than they would otherwise be.** Anything the query layer wants
to persist has to earn a route on the other service. That friction is mostly healthy, because it
forces the question of whether the thing should be persisted at all, but it is friction.

## What I would reconsider

**The idempotency gap was foreseeable and I missed it.** Making a write path cross a network
boundary means the caller can time out after the work succeeded. A client-supplied request
identifier, stored with the created record and checked before creating another, is a small
amount of work at design time and an unpleasant amount after you have duplicates in a customer's
system. I would build that into the route from the first version.

**Caller identity should have crossed the boundary too.** The route knows a request arrived; it
did not originally know which human was behind it. For an action that creates a record someone
will later be accountable for, that attribution belongs in the payload from the start.

I would make the same structural decision again. The single-writer rule is the reason data
investigations in this system were short.
