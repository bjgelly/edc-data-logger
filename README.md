# edc-data-logger

Replaces FactoryTalk Transaction Manager on the EDC gateway with a .NET bridge that reaches the
controller through micro-ADS (Werewolf Embedded) and calls the same SQL Server stored procedures. The
controller logic, procedures and database stay as they are. The controller already keeps every
record until the procedure's answer comes back, so the bridge holds no data of its own.

**Status: design.** No code yet.

- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md): the design and the reasons for it. What Transaction
  Manager actually does, the relay cycle, reading and writing safely through micro-ADS, type
  conversion, failures, the cutover from Transaction Manager, throughput, and the open decisions.
- [docs/INTERFACE-SPEC.md](docs/INTERFACE-SPEC.md): the concrete version. Transaction definitions
  generated from the Transaction Manager export, item URIs, types, the bridge cycle, SQL calls,
  configuration, telemetry (OpenTelemetry traces, metrics and logs), the 22 transactions, the
  shadow comparison and the lab checks.
- [docs/RING-BUFFER.md](docs/RING-BUFFER.md): how the EDC gateway's ring buffers, Transaction Manager
  and stored procedures work today, read from the controller printouts, the Transaction Manager
  export and the database project.
- `design/`: the source material: the Transaction Manager export (`EDC.xml`) and the controller
  printouts.
