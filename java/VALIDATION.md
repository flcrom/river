# Validation

The original port and API review were validated locally on macOS arm64 with
JDK 25, Maven, PostgreSQL 18, and the SQLite JDBC driver pinned in `pom.xml`.
The adapters also ran on JDK 27. The build now targets Java 21; the Java 21
compatibility trial is recorded below. Formatting works on JDK 21 and 25.

References:

- River Go/Rust: `eb16420fed22ce479f4843f0accd4c4bfba0885e` from
  `bg/plan-interoperable-rust-river-port`.
- River JS: `696c67b606c1202eee221f0718b15ee433260bdf` from
  `bg/interoperable-js-port`.

| Check | Result |
|---|---|
| Maven native tests and formatting | Passed, 112 library tests and 7 CLI tests |
| Insert-only profile | Passed |
| Full PostgreSQL profile, maintenance, resilience | Passed |
| SQLite storage/runtime and resilience | Passed |
| Go + Java + Rust + JS on PostgreSQL and SQLite | Passed |
| Same-host enqueue, worker, and mixed performance gate | Passed with the original throughput and p95 limits |
| Four-engine PostgreSQL soak | Passed, five minutes |
| Four-engine SQLite endurance | Passed, five repetitions of the upstream multi-engine suite |

The native tests include exact unique-key goldens, cron/maintenance goldens,
transaction rollback and hook atomicity, transactional completion, queue drain,
and filtering claims to registered job kinds. The shared harness exercises
cross-process state transitions, notifications, reconnection, leadership,
rescue, periodic insertion, cancellation races, migrations, and raw row equality.

`conformance/scenario-coverage.json` maps shared scenarios to their harness
implementations. `conformance/feature-inventory.json` preserves all 304 upstream
inventory entries and identifies their Java surfaces and API differences.
`conformance/reference-sources.json` records hashes of imported Go resources.

These results cover the pinned contracts and finite soak runs. They are not a
proof of every possible workload, nor a production deployment or a published
Maven release. The separate Pro report records a known Go-reference SQLite
failure; no Go source was changed to suppress it.

## Migration CLI validation

The CLI tests cover environment/argument handling, offline SQL export, dry runs,
targets and step limits, and launching the executable JAR without an external
classpath. Library tests cover concurrent SQLite initialization, migration
rollback, legacy migration history, and PostgreSQL schema creation and removal.
All were run with PostgreSQL enabled; no tests were skipped.

After the migration CLI changes, the PostgreSQL mixed conformance suite and both
SQLite storage/runtime suites passed against the pinned Go reference. The
README quick start was compiled and run; its remaining Java examples compiled,
and its SQLite testing example ran successfully.

## Public API review validation

The API review added regression coverage for typed retrieval and mixed-kind
batches, checked JDBC hook failure recovery, queue notification atomicity,
committed completion followed by a handler exception, and repeated graceful-stop
timeouts without implicit cancellation, and job-aware retry policies. All 112 library and 7 CLI tests passed
with PostgreSQL enabled and no skips; Maven's formatting checks passed.

The PostgreSQL maintenance, mixed-engine, and resilience suites and the SQLite
storage, runtime, and resilience suites were rerun against the pinned Go
reference. All README Java blocks compiled; the quickstart and SQLite JUnit
example executed successfully. The Rust/JS matrix, performance, and soak entries
above record the earlier port validation and were not repeated for this API pass.

## Java 21 compatibility trial

Validated on macOS arm64 with Temurin 21.0.12.1 and OpenJDK 25.0.1. On both
JDKs, the full Maven build and formatting checks passed: 114 library tests and
7 CLI tests, with PostgreSQL enabled and no skips. The library and executable
CLI target Java 21 (class file version 65), including when built on JDK 25.

The changes replace unnamed `_` variables with named parameters and use
`ReentrantLock` for queue and leadership operations that can block. Java 21
pins a virtual thread's carrier when it blocks inside a monitor. Two regression
tests launch a JVM with exactly one carrier and use latches to test blocking
claims and leadership callbacks. Both failed before the lock change and pass
after it. No public API or dependency changes were needed.

With both `JAVA_HOME` and `PATH` selecting JDK 21, the PostgreSQL maintenance,
mixed-engine, and resilience suites and the SQLite storage, runtime, and
resilience suites passed against the pinned Go reference. All README Java
blocks compiled on JDK 21; the quickstart and SQLite JUnit example executed.
The Java CI workflow now tests JDK 21 and 25 with its existing path filters.

Rust/JS peer, performance, and soak runs were not repeated for this trial;
the earlier results above do not establish their behavior on JDK 21.
