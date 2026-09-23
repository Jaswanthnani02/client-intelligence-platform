# What this repository is, and what it deliberately is not

This is an architecture case study. It documents design decisions and the reasoning behind them
for a production system I built and operated.

## Not included, on purpose

**No source code.** Not a file, not a function, not a snippet. Every code element referenced here
is described in prose.

**No schemas or field definitions.** The data model is described at the level of "one aggregate
document per client, partitioned by client identifier." The actual collections, fields and types
are not reproduced.

**No infrastructure identifiers.** No resource names, hostnames, endpoints, subscription or
tenant identifiers, region names or account names.

**No credentials, configuration or environment detail.** Including the names of environment
variables, which are themselves a map of the system's integration surface.

**No client information.** No names, no industries specific enough to identify one, no counts, no
revenue, no contract detail. Where an example is needed, it is invented.

**No security findings.** The system has an open internal audit with unremediated items, as any
real system does. None of it appears here, in any form, at any level of abstraction. A case study
describing where a live system is weak is not a case study, it is a disclosure.

**No business metrics.** No adoption numbers, time savings, cost figures or commercial outcomes.
Those belong to the employer, not to me, and quoting them publicly is their decision.

## Included

Architectural patterns, the tradeoffs considered, the reasoning behind each decision, the
failures encountered and what they changed. These are transferable engineering judgments rather
than proprietary assets.

Approximate structural figures — the number of scheduled jobs, of tools, of tests — appear where
they convey scale. They describe the shape of the system, not its data.

## On the employer

The system is described as belonging to "a managed service provider" rather than named. That is
deliberate: nothing here requires knowing which one, and a named employer invites readers to
attribute the opinions in this repository to them. The opinions are mine.

## Licence

The writing in this repository is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use the ideas freely; attribution is
appreciated.

The system it describes remains the property of the employer it was built for. This document
conveys no rights to it.

## If you think something here crosses a line

Open an issue or contact me, and I will remove it first and discuss it afterwards. The value of a
case study is not worth a disclosure argument.
