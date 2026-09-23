# What broke, and what it changed

Five production failures worth writing down. Each changed a practice, not just a line of code.

No dates, no identifiers, no client impact figures — see [NOTICE](../NOTICE.md). The engineering
is the point.

---

## A blocking call inside an async handler

**What happened.** The query service is asynchronous. A new code path introduced a synchronous
network call — a library doing its own HTTP fetch internally — inside an async request handler.
Under light load nothing was visible. Under real traffic the event loop stalled, because one
blocking call in a cooperative scheduler stops every concurrent request on that worker, not just
its own.

**Why it was hard to see.** It is not a crash. Latency degrades, requests queue, health checks
eventually fail, and the stack trace points at whatever timed out rather than at what blocked.
The cause and the symptom are in different places.

**What changed.** Every synchronous call in an async path is now wrapped to run off the event
loop. More usefully: this became a review checklist item, because it is invisible in a diff
unless you know the library's internals.

---

## A middleware that replayed a request body forever

**What happened.** A piece of middleware needed to inspect the request body and then pass it on.
The server interface delivers the body as a stream that can be consumed once, so the middleware
buffered it and replayed it downstream. The replay logic returned the buffered body on every
request for more data, rather than once followed by end-of-stream. Downstream kept asking. The
request never completed.

**Why it was hard to see.** It only manifested for requests with a body, on the path that used
that middleware, and it presented as a hang rather than an error.

**What changed.** A specific rule: **a shim that replays a consumed stream must deliver the
content once and then delegate.** More generally, middleware that sits between a protocol and its
handlers is high-risk code deserving disproportionate test coverage, because its failures look
like problems somewhere else.

---

## A fix verified against the wrong door

**What happened.** The service has two authentication paths: a service credential, and
directory-based sign-in for humans. A fix to the authorization layer was verified by scripting
the service-credential path, which is a single header and trivially automated. The
directory-based path remained broken. It was confirmed working and shipped.

**Why it was hard to see.** The verification genuinely passed. The mistake was in what was
verified, not how.

**What changed.** When a system has multiple authentication paths, **a fix is unverified until it
is verified on the path real users take** — even when that path is much harder to exercise. The
convenience of the easy path is exactly what makes it the wrong evidence.

---

## Every scheduled job stopped, silently, for weeks

**What happened.** The scheduled-job host authenticates to a storage account to manage its own
state. That credential became stale. The host could no longer coordinate, so it ran nothing. Not
one job fired.

**Why it was hard to see.** The failure was in the platform's own bookkeeping, below the
application. Application logs showed nothing, because no application code executed. There were no
errors because there were no runs. The only symptom was data quietly getting older, which looks
identical to a quiet period.

**What changed.** Two things. Monitoring moved from "did this job report an error" to **"did this
job report at all"** — absence of a signal became the alert, since a job that does not run cannot
report its own failure. And the first diagnostic step for stale data became checking the host's
own request telemetry before reading a line of application code.

---

## A schedule changed without its derived constant

**What happened.** A safety margin used in ingestion is derived from how often the job runs.
Someone changed the schedule. The constant did not move with it. The margin became too small for
the new interval, and a downstream recomputation stopped triggering correctly for roughly two
weeks.

**Why it was hard to see.** Both values were individually reasonable. Only the relationship
between them was wrong, and nothing expressed that relationship in a form anything could check.
A comment said to keep them in sync. The comment was right and was not read.

**What changed.** A test asserts the two agree, and fails the build otherwise. The wider practice:
**when one value is derived from another, make the derivation executable.** A documented
invariant is a hope. A tested invariant is a guarantee. Nearly every durable constraint in this
system ended up with a test behind it, and the ones that did not are the ones that broke.

---

## The pattern across all five

Four of these five were **silent**. Nothing crashed, nothing paged, no error appeared. The system
kept reporting success while doing less than it should.

That shaped how the platform is monitored more than any individual fix. The most valuable signals
turned out to be absence and staleness rather than errors: a job that did not report, a watermark
that has not moved, a quarantine collection that is growing. Systems that fail loudly are the easy
case. The expensive failures are the ones where everything looks fine.
