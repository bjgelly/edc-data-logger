# CLAUDE.md — EDC Data Logger (`edc-data-logger`)

This file provides guidance to Claude Code (claude.ai/code) when working in this repository. This
repository is the project's home: its context lives here and in the docs, not in any workspace-level
file elsewhere.

## What this is

A C#/.NET service, the **bridge**, that replaces FactoryTalk Transaction Manager on the EDC gateway
controller (`eAssy_EDC_Gway.ACD`). It polls each transaction's trigger through **micro-ADS** (Werewolf
Embedded, gRPC on `127.0.0.1:8081`), reads the bound tags, calls the same SQL Server stored procedure
Transaction Manager calls, writes any outputs, then writes the echo. The controller, the
procedures and the database are not changed (decided 6 Nov 2026; it replaced a 31 Oct draft
that had its own queue and database).

**Status: design. No code yet.**

## Read first

| Doc | Holds |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | The design and the reasons: relay rules, reading and writing order, types, failures, cutover, throughput, telemetry rationale (13.1), open decisions (15) |
| [docs/INTERFACE-SPEC.md](docs/INTERFACE-SPEC.md) | The concrete version: definitions generated from `EDC.xml`, item URIs, the cycle, SQL, configuration, telemetry (8), the 22 transactions, lab checks (12), open items (13) |
| [docs/RING-BUFFER.md](docs/RING-BUFFER.md) | How the controller, Transaction Manager and procedures work today, read from the printouts |
| `design/` | Source material: `EDC.xml` (Transaction Manager export, 2 Nov 2026) and the Logix printouts (PDF) |

## Rules the design depends on

Do not change any of these without reading the section that gives its reason:

- **The bridge holds no records and no state that must survive a restart.** The controller keeps
  each record until the echo matches (ARCHITECTURE.md, section 1).
- **One call in flight per transaction.** Read the trigger first and the data in a separate call.
  Write the data outputs, then the echo in a call of its own. Check each item's quality, not just the
  RPC status (sections 5 to 7).
- **Convert types as Transaction Manager did:** several INT and SINT members are read as unsigned
  (section 8).
- **Never let the bridge and Transaction Manager answer the same transaction** (section 10).
- **Polling, not subscriptions,** because of the micro-ADS subscription defects found on
  `andon-ads-mdi-gw` (section 11).
- **Telemetry is off by default, and a bad telemetry setting never stops the bridge**
  (INTERFACE-SPEC.md, section 8).

## Facts checked, and how

- Every count in INTERFACE-SPEC.md (22 enabled transactions, bound and output counts per transaction,
  type totals adding up to 1,612) was checked against `design/EDC.xml` with a parser on 6 Nov 2026.
  Re-run that check whenever `EDC.xml` is re-exported.
- micro-ADS behaviour comes from the digip-wiki
  (`C:\Work\source\repos\ra-ti-digital-platform\digip-wiki`): `services/microads/data-service.md`,
  `data-service-reference.md`, `concepts/cip-data-types.md`, `concepts/logger-lib.md`. Cite the page.
  A wiki statement is still not a lab result.
- In the docs, mark what is inferred or unmeasured as such. The main unmeasured item is micro-ADS
  throughput: the wiki's v1.0.5 figures suggest about 1 ms per item, which would make the 25 ms,
  44-item poll alone more than one micro-ADS can serve. Lab checks 4 and 5 settle it.

## Related repositories

- `C:\Work\source\repos\andon-ads-mdi-gw`: the first micro-ADS application (Node). It has the lab
  record of micro-ADS defects (power cycle, overlapping subscriptions), the Windows-service install
  pattern for micro-ADS (nssm, log clean-up) and `tools/probe.js`.
- `C:\Work\source\repos\andon-cgp-mdi-gw`: the CGP gateway that `andon-ads-mdi-gw` replaced.
  History only.

## Working here

- The design docs are the deliverable for now. Keep ARCHITECTURE.md and INTERFACE-SPEC.md consistent
  with each other: decisions in one, detail in the other, with cross-references by section number.
  Check those references when sections move.
- Telemetry follows Brian's `opentelemetry-dotnet.md` guide. It is not in this repo, so ask for it if
  it is needed.
- Commit only when asked.
