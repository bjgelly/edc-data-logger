# EDC Data Logger: Interface Specification

**Draft 0.2.** This is the concrete version of [ARCHITECTURE.md](ARCHITECTURE.md): the transaction
definitions, how they are generated, the bridge's cycle, its configuration and the lab checks.

The controller and the database are not changed, so their side of the interface is already fixed. It
is whatever Transaction Manager binds today (`design/EDC.xml`), and [RING-BUFFER.md](RING-BUFFER.md)
describes it. This document specifies the bridge's side.

Draft 0.1 specified a new controller queue, window protocol and database. It was withdrawn on
6 November 2026 (ARCHITECTURE.md, Purpose).

**Contents**

1. [What crosses each boundary](#1-what-crosses-each-boundary)
2. [Transaction definitions](#2-transaction-definitions)
3. [Item URIs](#3-item-uris)
4. [Types](#4-types)
5. [The bridge cycle](#5-the-bridge-cycle)
6. [SQL calls](#6-sql-calls)
7. [Configuration](#7-configuration)
8. [Telemetry: logs, traces and metrics](#8-telemetry-logs-traces-and-metrics)
9. [The 22 transactions](#9-the-22-transactions)
10. [Shadow comparison](#10-shadow-comparison)
11. [When the ladder or a procedure changes](#11-when-the-ladder-or-a-procedure-changes)
12. [Lab checks](#12-lab-checks)
13. [Open items](#13-open-items)

---

## 1. What crosses each boundary

| Boundary | What crosses | Defined by |
|---|---|---|
| Controller → bridge | Trigger and echo values (the poll); the bound tags of one transaction (the data read) | The ladder and the UDTs, unchanged |
| Bridge → SQL Server | One stored procedure call per trigger, with the same parameters Transaction Manager passes | The procedures in `edc-db-edc`, unchanged |
| SQL Server → bridge | Output parameters, the echo among them | The procedures |
| Bridge → controller | The data outputs (server exchanges only), then the echo | `EDC.xml` bindings |

---

## 2. Transaction definitions

### 2.1 Format

One JSON file holds every definition. Example, shortened (the cycle record has 277 bound tags):

```json
{
  "source": "EDC.xml, exported 2026-11-02",
  "transactions": [
    {
      "name": "EQRingBufferCycle01_StoreTM_wTransNumAck",
      "procedure": "dbo.spEQRingBufferCycle_StoreTM_wTransNumAck",
      "trigger": "EDC.RB_Cycle01.TransactionNumber",
      "echo": "EDC.RB_Cycle01.Interface",
      "parameters": [
        { "name": "@input_TMTransNum",  "source": "trigger", "sqlType": "int" },
        { "name": "@input_TMTimeStamp", "source": "localTime", "sqlType": "datetime" },
        { "name": "@input_ShopID",      "source": "constant", "value": 1, "sqlType": "int" },
        { "name": "@input_TransNum",    "source": "tag", "tag": "EDC_RB_Cycle01_Read.Header.TransNum", "tmType": "int16", "sqlType": "int" },
        { "name": "@input_Year",        "source": "tag", "tag": "EDC_RB_Cycle01_Read.TransTime.Year", "tmType": "uint16", "sqlType": "int" },
        { "name": "@input_Step000",     "source": "tag", "tag": "EDC_RB_Cycle01_Read.Data.Step[0]", "tmType": "uint16", "sqlType": "int" },
        { "name": "@output_SPResult",   "source": "tag", "tag": "EDC.RB_Cycle01.Interface", "tmType": "int32", "sqlType": "int", "direction": "inout" }
      ]
    }
  ]
}
```

The `sqlType` values above are illustrative; the generator takes them from the procedure.

| Field | Meaning |
|---|---|
| `trigger` | The tag polled for changes. |
| `echo` | The tag the last output is written to; always written last, in its own call. |
| `parameters` | In the procedure's order. |
| `source: trigger` | The polled trigger value, N. Not read again. |
| `source: tag` | Read from the controller in the data read. |
| `source: constant` | `value`, as given. |
| `source: localTime` | The bridge host's local time at the call. |
| `tmType` | The type Transaction Manager reads the tag as. It sets the conversion (section 4). |
| `direction: inout` | An output. Its tag is read and passed in, then written back from the output value. The default is `in`. |

### 2.2 Generating them from EDC.xml

A small console tool in this repo, `TmImport`, reads `EDC.xml` and the procedures' parameter
metadata (`sys.parameters`), and writes the definitions file. The output is checked into the repo
and reviewed like code. The tool **fails** on anything it does not recognize, rather than guessing:

| `EDC.xml` | Becomes | Fails if |
|---|---|---|
| `transaction` with `collection_status` `enabled` | a definition | — (disabled ones are skipped: the four Track buffers) |
| `unscheduled_trigger_name` → its `data_point` → `item` | `trigger` | the trigger type is not `trigger_on_change` with a delta of 1 |
| `update_flag` | — | it is not `realtime_always` |
| `bind`, type `datapoint`, direction `input` | `source: tag`, or `source: trigger` if its tag is the trigger | the data point's item is not on the `[EDC]` shortcut |
| `bind`, type `datapoint`, direction `input_output` | `source: tag`, `direction: inout` | — |
| `bind`, type `constant` | `source: constant` | — |
| `bind`, type `expr_trantimeof`, expression `TransTimestamp("local")` | `source: localTime` | any other expression |
| `bind`, type `null` (`@RETURN_VALUE`) | dropped | — |
| the last `input_output` bind | its tag is `echo` | its name is not `@output_SPResult` or `@output_TransNum` |
| `data_point` `data_type` | `tmType` | a type outside section 4 |
| — | `sqlType` from `sys.parameters` | a bound parameter is missing from the procedure, or the procedure has a parameter with no binding |

A data point's `item` gives the tag: `/EDC::[EDC]EDC.ServerCfg.DataRsp.Name` is the tag
`EDC.ServerCfg.DataRsp.Name`. Array indices are written without leading zeros: the item
`LDD_LineStateSend[00]` becomes `LDD_LineStateSend[0]` (lab check 3).

All 26 transactions in `EDC.xml` were checked against these rules on 6 November 2026: the 22 enabled
ones fit, and in every one the echo is the last `input_output` bind.

---

## 3. Item URIs

The route is the gateway controller's path, as micro-ADS needs it:

```text
cip.name://route:<route>/<tag>
cip.name://route:2:192.168.2.236/1:0/EDC_RB_Cycle01_Read.Data.Step[0]
```

Every member and array element is its own item. Never read a structure or a whole array as one
item, and never use the `,L<n>` element-count modifier (ARCHITECTURE.md, section 8).

---

## 4. Types

| `tmType` | Logix type | micro-ADS read | Passed to SQL as | Written back as | Bound in the 22 |
|---|---|---|---|---|---|
| `int32` | DINT | `int32_value` | `int` | DINT | 384 |
| `int16` | INT | to confirm (lab check 2) | `short` | INT | 348 |
| `uint16` | INT | to confirm | `(ushort)(short)value` | — (inputs only) | 496 |
| `int8` | SINT | to confirm | `sbyte` | — (inputs only) | 34 |
| `uint8` | SINT | to confirm | `(byte)(sbyte)value` | — (inputs only) | 296 |
| `bool` | BOOL | `bool_value` | `bit` | BOOL | 40 |
| `single_flt` | REAL | `float_value` | `real` | REAL | 5 |
| `string` | STRING | `string_value` | `nvarchar`/`varchar` per the parameter | STRING | 9 |

The value is converted by `tmType` first, then given to SQL Server with the parameter's own type.
The cycle VIN parameters are `VARBINARY`: each `uint8` becomes a one-byte array.

Written-back types occur only in the server exchanges: DINT, INT, BOOL, REAL and STRING (lab
check 8).

---

## 5. The bridge cycle

### 5.1 The poll

Every `PollMs` (25 ms), one `ReadItems` call reads the trigger and the echo of every transaction that
is `Live` or `Shadow`: 44 items when all 22 are on.

- **A trigger that differs from the last value handled** signals that transaction's worker. A worker
  that is busy takes the latest value when it finishes; intermediate values are skipped, as the
  controller has already moved past them.
- **An echo that changes to a value the bridge did not write**, on a `Live` transaction, means
  another client is answering. Log it, raise the alarm and stop that worker (ARCHITECTURE.md,
  section 10).
- **A BAD item** is skipped for that poll.

### 5.2 One worker cycle

For a `Live` transaction, with N the trigger value that signalled:

1. **Read.** One `ReadItems` call for every `source: tag` parameter. If any item is BAD, wait
   `RetryReadMs` and read again, unless the trigger has moved on, in which case start again with the
   new N.
2. **Convert** each value by `tmType` (section 4).
3. **Call** the procedure (section 6). On any failure, log it, wait out the backoff, and end the cycle
   **without writing anything**.
4. **Write data outputs,** if the transaction has any besides the echo: one `WriteItems` call. Every
   item must be GOOD; a BAD item is retried once. If it is still BAD, end the cycle without writing
   the echo.
5. **Write the echo**, the echo parameter's output value, in its own `WriteItems` call. Retry once
   if BAD.
6. **Record N as handled,** and the echo value as written by the bridge.

A ring buffer cycle is two micro-ADS calls (one read, one write); a server exchange cycle is three.

### 5.3 Shadow mode

For a `Shadow` transaction, the worker does steps 1 and 2, then reads the trigger again. If it is
still N, it appends the parameter set to the shadow file (section 10). If not, Transaction Manager
has already answered and the controller may have staged the next record, so the read is discarded.
Shadow mode never calls a procedure and never writes to the controller.

### 5.4 Start-up and reconnect

1. Load and validate the definitions, and check each procedure exists.
2. Connect to micro-ADS; on `UNAVAILABLE`, retry with bounded backoff.
3. The first poll sets each transaction's starting point. If its echo equals its trigger, N is
   already answered. If not, the worker is signalled and answers it. For a ring buffer that is
   harmless even if the record was stored before the restart, because the procedure recognizes the
   repeat.

After a lost connection, the bridge starts again at step 2. It carries nothing over.

---

## 6. SQL calls

- `Microsoft.Data.SqlClient`, `CommandType.StoredProcedure`, one pooled connection per call, opened
  and closed around it. No outer transaction.
- Parameters typed from `sqlType`. Output and in/out parameters use `Direction = InputOutput`, as
  Transaction Manager binds them.
- `CommandTimeout` is `SqlTimeoutSeconds` (15). The controller retries after 5 s anyway; the timeout
  only frees the worker.
- Any `SqlException`, including error 266 (transaction count mismatch), is a failed call: nothing is
  written. Returning the connection to the pool resets it, which rolls back a transaction the
  procedure left open (RING-BUFFER.md, section 8.1).
- Backoff after a failure doubles from 1 s to `SqlBackoffMaxSeconds` (30), per transaction, and resets
  after a success.
- Connection string from the environment: `ConnectionStrings__EDC`. Its login needs `EXECUTE` on the
  22 procedures, and `VIEW DEFINITION` only for whoever runs `TmImport`.

---

## 7. Configuration

`appsettings.json`:

```json
{
  "MicroAds": {
    "Endpoint": "http://127.0.0.1:8081",
    "Route": "2:192.168.2.236/1:0"
  },
  "Relay": {
    "DefinitionsPath": "definitions/EDC.json",
    "PollMs": 25,
    "RetryReadMs": 100,
    "SqlTimeoutSeconds": 15,
    "SqlBackoffMaxSeconds": 30,
    "ShadowPath": "shadow",
    "Transactions": {
      "EQRingBufferCycle02_StoreTM_wTransNumAck": "Live",
      "EQRingBufferFault06_StoreTM_wTransNumAck": "Shadow"
    }
  },
  "Telemetry": {
    "Exporter": "none",
    "OtlpEndpoint": "http://127.0.0.1:4317",
    "ServiceName": "edc-bridge",
    "Site": "",
    "ControllerStatsSeconds": 10,
    "CaptureRecordValues": false
  }
}
```

- **`Telemetry.Exporter`** is `none` (the default), `console` or `otlp` (section 8). It is
  validated apart from the rest: a bad value leaves telemetry off and the bridge running.

- **Each transaction is `Off`, `Shadow` or `Live`.** Anything not listed is `Off`. Moving a
  transaction to `Live` is a cutover step: it is disabled in Transaction Manager first
  (ARCHITECTURE.md, section 10).
- **The SQL connection string comes from the environment, never the file.**
- **`Route`** is the gateway controller's path (open item 1).

---

## 8. Telemetry: logs, traces and metrics

The bridge uses OpenTelemetry for traces, metrics and logs, following the team's guide
(`opentelemetry-dotnet.md`). **It is off by default.** With it off, the bridge writes only its rolling
log file, and that file must be enough to run it. Turning it on adds no reads from the controller
except the slow diagnostics read in section 8.6.

### 8.1 What it has to answer

| Question | Answered by |
|---|---|
| Is the bridge keeping up with each ring buffer? | The controller's own `RecordCount` (8.6), and cycle rate and duration (8.5) |
| Is micro-ADS saturated? (ARCHITECTURE.md, section 11) | Poll interval against `PollMs`, poll overruns, and micro-ADS call time by kind (8.5) |
| Where did one slow or failed cycle spend its time? | The cycle trace (8.3) |
| How often does the controller have to resend a record, and why? | `app.record.repeat` on the cycle span, and the repeat and echo counters |
| Is anything being lost? | `WriteBlockErrCnt` per buffer (8.6). Loss happens upstream of the bridge, but this is where it becomes visible over time |
| Is another client answering? | The foreign-echo counter, and the Error log line |

### 8.2 Registration

- Its own `Telemetry` configuration section (section 7), kept apart from the `Relay` and
  `MicroAds` settings that must be valid for the bridge to start. **A bad telemetry setting logs a
  warning and leaves telemetry off. It never stops the bridge.** An exporter value that is not
  recognized is logged by name.
- At start-up the bridge logs what was chosen: on or off, exporter, endpoint, and whether record
  values are captured (8.9).
- One `ActivitySource` and one `Meter`, both named `EdcDataLogger.Bridge` after the assembly.
- Resource: `service.name` = `edc-bridge`, `service.version` from the assembly, `service.instance.id`
  = a new GUID per start, and `app.site` from configuration (for example `TMMI EDC`), so that bridges
  at several shops can share one backend. Not the host name.
- `WithTracing`, `WithMetrics` and `WithLogging`, each with the OTLP exporter. The console exporter is
  for tracing only.
- **No ASP.NET Core, HttpClient or gRPC client instrumentation.** The bridge has no inbound HTTP. Its
  gRPC calls are traced by hand (8.3), for two reasons. micro-ADS returns `Ok` for a call whose items
  are all BAD, so an automatic span would show a failed read as a success. And automatic
  instrumentation would also trace every poll, 40 times a second, as a trace of its own.
- SQL: the `OpenTelemetry.Instrumentation.SqlClient` package if its release status suits (it has
  shipped on its own pre-release line; check the version current at build time). Otherwise a span
  written by hand. Never both.
- All OpenTelemetry packages are pinned to one version. The guide uses 1.20.1.

### 8.3 Spans

One trace per handled trigger value. That is the unit people will ask about: "what happened to
Fault06's record under N = 4512?".

| Span | Kind | One per | Parent |
|---|---|---|---|
| `relay.cycle` | Internal | worker cycle (section 5.2 or 5.3) | root |
| `microads.read` | Client | `ReadItems` attempt: the data read, or shadow mode's second trigger read | `relay.cycle` |
| `sql.execute` | Client | procedure call (from the SqlClient instrumentation, or by hand) | `relay.cycle` |
| `microads.write` | Client | `WriteItems` attempt: data outputs, or the echo; a retry is a second span | `relay.cycle` |

No span for the poll (8.5 covers it), the conversions, the definitions load, or anything else that
runs every 25 ms or per item.

**Status.** The cycle span is `Error` when the outcome is `sql-failed`, `read-bad` or `write-bad`, or
when the procedure returned -N or 0. Not for `answered` (including a record acknowledged without
being stored, such as a year before 2020), `superseded`, `shadow` or `shadow-discarded`: those are
the system working as designed. Exceptions are recorded with `AddException` on the span where they
happened.

### 8.4 Attributes

All under `app.`; the instrumentation libraries' own attributes pass through unchanged.

| Attribute | On | Values |
|---|---|---|
| `app.tx.name` | cycle | the 22 transaction names |
| `app.tx.mode` | cycle | `live`, `shadow` |
| `app.tx.n` | cycle | the trigger value N |
| `app.cycle.outcome` | cycle | `answered`, `sql-failed`, `read-bad`, `write-bad`, `superseded` (the trigger moved on during BAD-read retries), `shadow`, `shadow-discarded` |
| `app.cycle.echo_kind` | cycle | `ack` (+N), `resend` (-N), `nomatch` (0), `none` (nothing written) |
| `app.cycle.path` | cycle | the steps, joined with `>`: `read>sql>echo`, `read>read>sql>data>echo`, `read>superseded` |
| `app.cycle.triggers_skipped` | cycle | trigger values that passed while the worker was busy (section 5.1) |
| `app.record.eqid`, `.trans_num`, `.cmd_num` | cycle (ring buffers, `LddLineStatus`) | from the record header |
| `app.record.keepalive` | cycle | `cmd_num` is not 0 (RING-BUFFER.md, section 3.2) |
| `app.record.repeat` | cycle | the record has the same duplicate key as this transaction's previous cycle, so the controller resent it (RING-BUFFER.md, section 8) |
| `app.read.kind` | read | `data`, `trigger-recheck` |
| `app.write.kind` | write | `data`, `echo` |
| `app.attempt` | read, write | 1, 2, … |
| `app.items.count`, `app.items.bad` | read, write | item counts |
| `app.items.bad_quality` | read, write | the first BAD item's quality enum name |
| `app.sql.error_number` | sql | for example 266 |

`app.cycle.outcome` is derived in one place, from the worker's result type, and a test pins the
mapping, so a new result cannot fall silently into the wrong bucket. `app.cycle.path` is what tells a
clean cycle from one that read three times before it succeeded: both show `answered`.

The record header fields identify a record without carrying its content, so they are always on (8.9).

### 8.5 Metrics

Complete and unsampled. Attributes on metrics are limited to the bounded sets below. Never N, EQID or a
record field.

| Instrument | Type, unit | Attributes | Measures |
|---|---|---|---|
| `edc_bridge.cycles` | counter | `tx`, `mode`, `outcome` | worker cycles |
| `edc_bridge.cycle.duration` | histogram, ms | `tx`, `outcome` | from the poll that saw N to the end of the cycle |
| `edc_bridge.echoes` | counter | `tx`, `echo_kind` | the procedure's answers |
| `edc_bridge.triggers.skipped` | counter | `tx` | values passed over while busy |
| `edc_bridge.records.repeated` | counter | `tx` | resends the controller made |
| `edc_bridge.records.keepalive` | counter | `tx` | keep-alive records relayed |
| `edc_bridge.poll.interval` | histogram, ms | — | start of one poll to the start of the next; above `PollMs` means micro-ADS cannot keep pace |
| `edc_bridge.poll.overruns` | counter | — | polls that took longer than `PollMs` |
| `edc_bridge.microads.duration` | histogram, ms | `call` (`poll`, `read`, `write`), `result` (`good`, `bad`, `rpc-error`) | every micro-ADS call |
| `edc_bridge.microads.items.bad` | counter | `call`, `quality` | BAD items, by quality |
| `edc_bridge.sql.duration` | histogram, ms | `tx`, `result` | procedure calls |
| `edc_bridge.sql.failures` | counter | `tx`, `kind` (`timeout`, `txcount-266`, `connect`, `other`) | failed calls |
| `edc_bridge.echoes.foreign` | counter | `tx` | echoes the bridge did not write |
| `edc_bridge.transaction.state` | gauge | `tx`, `state` (`off`, `shadow`, `live`, `stopped`) | 1 for the current state |
| `edc_bridge.microads.connected` | gauge | — | 1 or 0 |

Histograms use explicit boundaries of 1, 2, 5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000 and
15000 ms, set with a View: the poll and the echo write sit in the first few buckets, and the SQL
timeout in the last.

Nothing is written to a SQL table (open item 3 is answered by this).

### 8.6 Controller diagnostics

The controller already counts what matters most: each ring buffer's backlog and the messages it
dropped (RING-BUFFER.md, sections 2.5 and 9). Today those counts are visible only on the HMI, and
only as current values. While telemetry is on, the bridge reads them every `ControllerStatsSeconds`
(10 s), in a `ReadItems` call of their own, and publishes them as gauges:

| Gauge | From, per ring buffer (`buffer` attribute: the 16 buffer names) |
|---|---|
| `edc_controller.rb.record_count` | `RecordCount`: records waiting, including the one in flight |
| `edc_controller.rb.record_count_max` | `RecordCountMax` |
| `edc_controller.rb.write_block_errors` | `WriteBlockErrCnt`: messages dropped because the buffer was full. Reset by hand, so it is a gauge, and the alert is any increase |
| `edc_controller.rb.shift_count` | `ShiftCnt` |
| `edc_controller.rb.alarm` | `BufferAlarm` |
| `edc_controller.release_fast` | `ServerStat_TMokTMR.DN`, once: 1 while the ring buffers release with no delay (RING-BUFFER.md, section 5.1) |

That is 81 items every 10 s, a small load next to the poll. A rising `record_count` is the clearest
sign that the bridge is falling behind, and it is what lab check 5 measures. The tag paths are
inferred from the printouts (`EDC.RB_Fault06.RecordCount`, and the timer in program
`ServerInterface`). Lab check 11 confirms them and their external access.

### 8.7 Logs

- **The rolling file stays,** at every setting, as the primary record.
- **State changes at Information or above, not repetitions.** These are: start-up and the definitions
  loaded; each transaction's mode; micro-ADS connected and lost; a transaction starting to fail and
  recovering, with the count of failures in between; a worker stopped by a foreign echo (Error). A
  controller outage would otherwise write a line per transaction every `RetryReadMs`: 220 lines a
  second.
- **One line per cycle at Debug** (transaction, N, outcome, echo, step times), to the local file only.
- A logging scope per cycle carrying the transaction and N, so every line written inside a cycle
  carries them and its trace ID.
- Over OTLP at Information, with `IncludeFormattedMessage` and `IncludeScopes` on. No console log
  exporter.

**micro-ADS's own log** holds what the bridge cannot see: the CIP status behind a BAD item (535, 542,
543; wiki, *Reading the write result*). Shipping it beside the bridge's logs (8.8) lets a BAD
write span be matched to its cause by time. There is no trace ID to join on.

**micro-ADS's own OTLP export** is compiled in but off in every bundle. Enabling it means an `otlp`
appender in micro-ADS's `Logger/default.yaml`, and a missing `endpoint` or `service_name` there
makes micro-ADS panic at start-up (wiki, `concepts/logger-lib.md`). Its traces start afresh rather
than continuing the bridge's: the wiki finds trace context is not continued inside micro-ADS.
Whether its gRPC server reads an incoming `traceparent` has not been checked. Use it, if at all, as a
lab experiment, not in production.

### 8.8 Export and sampling

```text
  BRIDGE HOST (Windows)                                OBSERVABILITY HOST
  bridge ──OTLP──► Alloy 127.0.0.1:4317 ──push──►  Tempo (traces), Loki (logs),
  micro-ads\logs\ ──file tail──► Alloy              Prometheus (metrics), Grafana
```

- **Alloy on the bridge host,** receiving OTLP on loopback and pushing outward. It is needed here for
  two jobs: tail sampling, and tailing micro-ADS's log files. Nothing on the host listens to the
  network for it.
- **Volume.** At about 45 cycles a second and about 4 spans each, the bridge produces about 180
  spans a second, or 15 million a day. All of them go to Alloy over loopback. Alloy keeps every trace
  that is `Error`, has an outcome other than `answered`, has `app.record.repeat`, or is slower than
  `SlowCycleMs` (250 ms to start with), plus 1% of the rest. In shadow mode and in the lab, it keeps
  everything.
- **Without Alloy,** the bridge exports straight to the backend with head sampling
  (`ParentBased(TraceIdRatio)`). That loses rare failures from the traces; the metrics still count them.
- **Telemetry never holds up a cycle.** Export is batched and asynchronous. When the queue is full,
  spans are dropped, and the relay does not wait.
- **The destination in production is open** (open item 7). The Digital Platform's LGTM stack is
  cloud-only, with no edge pattern (wiki, `services/observability/lgtm-stack.md`). Ask what the plant
  already runs before choosing.

### 8.9 What is not recorded

Record values (the 23 to 277 parameters) are not put on spans or in logs. A switch,
`Telemetry:CaptureRecordValues` (off by default, logged at start-up), adds them to the cycle span,
each truncated to 1 KB and marked when truncated. The cycle record carries a VIN, which is a vehicle
identifier. Whether it may leave the plant in telemetry is the data owner's decision (open item 8).
The shadow file (section 10) holds the same data, so the same decision covers it.

### 8.10 Tests

- An `ActivityListener` collects real spans in unit tests, and one test prints the span tree with its
  attributes (reading `TagObjects`, not `Tags`). A `MeterListener` does the same for the instruments.
- The outcome classification has a test that runs every worker result through it.
- Run with the console exporter: a cycle produces a `relay.cycle` span with its children. Run with no
  `Telemetry` section: zero spans, and the "telemetry is off" line.
- Shadow mode in the lab against the Aspire dashboard, before anything else is decided (lab check 13).

---

## 9. The 22 transactions

From `EDC.xml`. "Bound tags" counts the data points, including the trigger and the in/out tags; the
data read is one fewer, since the trigger comes from the poll. "Data outputs" excludes the echo.

| Transaction | Trigger | Echo | Procedure | Bound tags | Data outputs | Phase |
|---|---|---|---|---|---|---|
| `EQRingBufferFault01`–`06_StoreTM_wTransNumAck` | `EDC.RB_Fault0n.TransactionNumber` | `EDC.RB_Fault0n.Interface` | `spEQRingBufferFault_StoreTM_wTransNumAck` | 23 each | 0 | 1 |
| `EQRingBufferQuality01`–`04_StoreTM_wTransNumAck` | `EDC.RB_Quality0n.TransactionNumber` | `EDC.RB_Quality0n.Interface` | `spEQRingBufferQuality_StoreTM_wTransNumAck` | 71 each | 0 | 1 |
| `EQRingBufferTrend01`–`02_StoreTM_wTransNumAck` | `EDC.RB_Trend0n.TransactionNumber` | `EDC.RB_Trend0n.Interface` | `spEQRingBufferTrend_StoreTM_wTransNumAck` | 142 each | 0 | 1 |
| `EQRingBufferSummary01`–`02_StoreTM_wTransNumAck` | `EDC.RB_Sum0n.TransactionNumber` | `EDC.RB_Sum0n.Interface` | `spEQRingBufferSummary_StoreTM_wTransNumAck` | 44 each | 0 | 1 |
| `EQRingBufferCycle01`–`02_StoreTM_wTransNumAck` | `EDC.RB_Cycle0n.TransactionNumber` | `EDC.RB_Cycle0n.Interface` | `spEQRingBufferCycle_StoreTM_wTransNumAck` | 277 each | 0 | 1 |
| `LddLineStatus` | `LDD_LineStateTransNum` | `LDD_LineStateTransNumAck` | `spLineStatusStoreTM` (LDD schema) | 33 | 0 | 2 |
| `SysProcStatusBacklog_SendTM_wTransNumAck` | `EDC.ServerStat.Ctl.TransactionNumber` | `EDC.ServerStat.Ctl.Interface` | `spSYSProcStatusBacklog_SendTM_wTransNumAck` | 10 | 8 | 3 |
| `EQHealthStatus_SendTM_wTransNumAck` | `EDC.ServerHealth.Ctl.TransactionNumber` | `EDC.ServerHealth.Ctl.Interface` | `spHealthStatus_SendTM_wTransNumAck` | 5 | 3 | 3 |
| `CFG_Download_wTransNumAck` | `EDC.ServerCfg.Ctl.TransactionNumber` | `EDC.ServerCfg.Ctl.Interface` | `spCFG_Download_wTransNumAck` | 48 | 23 | 3 |
| `CFG_PdmDownload_wTransNumAck` | `EDC.ServerPdm.Ctl.TransactionNumber` | `EDC.ServerPdm.Ctl.Interface` | `spCFG_PdmDownload_wTransNumAck` | 134 | 66 | 3 |
| `CFG_DownloadIgnoreMask_wTransNumAck` | `EDC.ServerMask.Ctl.TransactionNumber` | `EDC.ServerMask.Ctl.Interface` | `spCFG_DownloadIgnoreMask_wTransNumAck` | 34 | 16 | 3 |

Every ring buffer and `LddLineStatus` also passes `@input_TMTimeStamp` (local time) and
`@input_ShopID` (1). The server exchanges pass `@input_ShopID` only.

---

## 10. Shadow comparison

The shadow file is one JSON line per record: transaction, N, the bridge's time, and every parameter
name and value as it would have been passed.

For the ring buffers, a comparison script matches each line to the row Transaction Manager stored,
by `EQID`, `Transaction_Num`, `Command_Num` and the event time built from the `TransTime` parameters
(the procedure's own duplicate key; RING-BUFFER.md, section 8). It then compares every stored column.
A record counts as matched only if every value is equal.

Records to make sure the test period includes:

- steps or durations above 32,767, and SINT values above 127 (the unsigned conversions);
- a VIN (the `VARBINARY` handling);
- keep-alive records (`Command_Num` not 0);
- a record with a year before 2020 or a bad date, which the procedure acknowledges without storing.

The gateway's test sources (RING-BUFFER.md, section 2.7) can generate fault and quality records on
demand in the lab.

---

## 11. When the ladder or a procedure changes

The definitions follow `EDC.xml`, so the process is the one already used for Transaction Manager:

1. Change the ladder or the procedure, and update the Transaction Manager configuration as today.
2. Export `EDC.xml` again and run `TmImport`.
3. Review the change to the definitions file, and deploy it with the bridge.

`TmImport` fails if a procedure and its bindings disagree (section 2.2), so a mismatch is caught
before deployment, not in production. Once Transaction Manager is retired, the definitions file
becomes the master, and changes are made to it directly.

---

## 12. Lab checks

Against the lab controller, with a copy of the gateway program and a development copy of the EDC
database. `probe.js` (`andon-ads-mdi-gw\tools`) covers the reads. Writes need a small .NET console in
this repo, which is also the first piece of the bridge.

**Before shadow mode (phase 1):**

| # | Check | Settles |
|---|---|---|
| 1 | The 23 tags of `EDC_RB_Fault06_Read` and the trigger read GOOD, with the values Studio 5000 shows | URI form, nested members |
| 2 | Which micro-ADS value field INT and SINT arrive in, for negative values and values above 32,767 and 127 as Logix shows them | The conversions in section 4 |
| 3 | `Tag[00]` and `Tag[0]`: which forms micro-ADS accepts | Index normalization in `TmImport` |
| 4 | Time of: the 44-item poll; reads of 23, 71, 142 and 277 items; a one-item write. Repeated, with distributions | ARCHITECTURE.md, section 11 |
| 5 | **Sustained capacity:** the fault pattern (poll, 23-item read, one-item write) in a loop, raising the rate until micro-ADS saturates, with other transactions' polls running | One micro-ADS or several |

**Before going live (phase 1):**

| # | Check | Settles |
|---|---|---|
| 6 | `WriteItems` to `RB.Interface` returns GOOD, and the controller releases the record | Write access, the echo |
| 7 | The first write after a download returns BAD (535) once, then GOOD | The retry-once rule |

**Before the server exchanges (phase 3):**

| # | Check | Settles |
|---|---|---|
| 8 | `WriteItems` of STRING, BOOL, REAL and INT values lands correctly | The written-back types |
| 9 | In `ServerInterface`, each of the five exchanges sets its request before changing its number, and holds it until answered | Read safety for the exchanges (ARCHITECTURE.md, section 6) |
| 10 | The Transaction Manager server's time zone | `@input_TMTimeStamp` |

**Telemetry and recovery (during phase 1):**

| # | Check | Settles |
|---|---|---|
| 11 | The controller diagnostic tags in section 8.6 read GOOD under the paths given there | The diagnostics read |
| 12 | **Controller power cycle while the bridge is reading** (shadow mode is enough): every read goes BAD, then recovers on its own once the controller is back. Afterwards each item carries its own value: check that the EQID, TransNum and VIN fields of a record are mutually consistent. The test runs once with an outage under a minute and once with one over 5 minutes. | Whether `ReadItems` has the defect seen with subscriptions on `andon-ads-mdi-gw` (ARCHITECTURE.md, section 9) |
| 13 | Shadow mode exporting to the Aspire dashboard: cycle traces, the metrics in section 8.5 and the diagnostics in 8.6 all arrive. Then with no `Telemetry` section: zero spans | Section 8 end to end; the default-off claim |

---

## 13. Open items

| # | Item | Owner |
|---|---|---|
| 1 | The gateway controller's route (IP address and slot), and the host for micro-ADS and the bridge | Controls / IT |
| 2 | The SQL login for the bridge, and its permissions | DBA |
| 3 | ~~Metrics: log only, or a SQL table too~~ Answered by section 8: OpenTelemetry, no SQL table | — |
| 4 | micro-ADS licensing material for the production host | Digital Platform |
| 5 | Who disables Transaction Manager transactions during cutover, and under which change process | Operations |
| 6 | Whether to fix the procedures' missing rollback (ARCHITECTURE.md, section 15) | DBA |
| 7 | Where telemetry goes in production: a Grafana stack, or what the plant already runs. Whether a bridge host may push to it | Toyota IS / operations |
| 8 | Whether record values, the VIN in particular, may be captured in telemetry and kept in the shadow file (section 8.9) | Data owner (Toyota) |
