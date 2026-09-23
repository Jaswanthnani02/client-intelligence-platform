# 0005 — Authorize every call, on both role and tenant

**Status:** Accepted.

## Context

The query service exposes the platform to a language model as callable tools. Two kinds of caller
reach it: automated systems using a service credential, and people authenticated through the
organization's directory.

Two properties of the deployment ruled out the usual approach of resolving permissions once and
caching them for a session. The service runs stateless across several replicas, so consecutive
calls from one person may land on different instances with no shared memory. And the caller is a
language model choosing tools autonomously — the sequence of calls is not a flow anyone
designed, so there is no natural point at which "the session began."

What the service guards is the material question. It holds a continuously refreshed mirror of an
entire book of business: service history, documentation, contact identities, financial records.
An authorization defect is a cross-client data exposure.

Two independent things must be true for a call to proceed. The caller must be **allowed to use
that capability**, and allowed to see **that particular client**. They are genuinely independent
— a senior person may use every tool but be scoped to their own accounts, and a junior person may
see many accounts through a narrow set of tools.

## Decision

Authorize **every tool invocation**, against both axes, with no caching of the decision.

Callers arrive through one of two doors: a service credential, or a bearer token validated
against the directory. When neither is configured, the service **fails closed** and rejects
everything.

Human callers carry a role derived from directory group membership, and a per-user list of
permitted clients. The role governs which tools; the list governs which clients. Both are checked.

The load-bearing detail: **a tool not explicitly assigned to a permission tier is restricted to
the highest tier.** Adding a tool and forgetting to configure its access produces a tool nobody
below administrator can call — an obvious, reported, harmless failure. The opposite default
produces a tool everybody can call, which fails silently and is discovered by audit or by
incident.

## Consequences

**Permission changes take effect on the next call**, because nothing is cached. Revoking access
is immediate rather than eventually.

**No session state**, so replicas scale freely and a restart loses nothing.

**The failure direction is chosen.** A configuration mistake under-grants. That is the correct
direction for a system holding other organizations' data.

**Per-call checks cost latency.** Group membership resolution is cached at the service level with
a short lifetime — a compromise between correctness and per-call directory lookups. Token
signature verification uses cached signing keys.

**A subset of capabilities is cross-client by design.** Portfolio-level views answer questions
spanning all clients, so they cannot be narrowed by a per-client list. Those are enumerated
explicitly rather than falling out of the logic, and documented as an exception. This is the part
of the model that most needs to be stated plainly: "this user is scoped to three clients" is true
of the per-client tools and false of the portfolio ones.

## What I would reconsider

**The portfolio exception deserves its own tier.** It is currently an enumerated exception inside
the role model, which means understanding a user's real access requires reading both the role and
the exception list. A distinct permission representing "may see cross-client aggregates" would be
clearer and harder to grant by accident.

**Registration should have been mechanically enforced from the start.** The rule that a new tool
must be assigned a tier, and documented, was upheld by code review. Review catches this most of
the time. A test that enumerates the registered tools and asserts each appears in exactly one
tier and in the documentation catches it every time, and I eventually wrote one — after
discovering by inspection that the enforcement had been manual all along.

**Verify against the path real users take.** A fix to this layer was once confirmed against the
service-credential door, which is trivially scriptable, while the bearer-token door remained
broken. The easy path to test and the path users take were not the same path. That is now a rule
I apply everywhere: when a system has two authentication paths, a fix is unverified until it is
verified on the one people actually use.
