# Mission: Migrate CardDemo from COBOL/CICS/VSAM to a Java Application

## Goal

Migrate the CardDemo mainframe credit card management application in this repository
to a modern Java application that preserves current business behavior. The end state
is a runnable, tested Java application covering the core online transactions and
batch jobs, loadable from the existing sample data, with a documented one-command
startup and an automated regression suite.

## Current state (inventory)

| Artifact | Count | Notes |
| :--- | ---: | :--- |
| COBOL programs | 44 (~30,175 lines) | CICS online + batch; largest are COACTUPC, COCRDUPC, COCRDLIC, COCRDSLC (~180-70 KB each) |
| Copybooks | 62 (~10,650 lines) | Includes REDEFINES, OCCURS, COMP/COMP-3 layouts |
| JCL jobs | 46 (~2,930 lines) | File loads, POSTTRAN, INTCALC, COMBTRAN, TRANBKP, CREASTMT, TRANIDX, export/import |
| BMS mapsets | 21 (~5,756 lines) | One per online screen family |
| Assembler | 2 | MVSWAIT, COBDATFT (date conversion) |
| Sample data | 12 datasets | EBCDIC and ASCII copies under `app/data/` |
| Schedulers | 2 | Control-M and CA7 definitions |

Data stores: VSAM KSDS with alternate indexes (primary), optional Db2, IMS DB, and MQ
modules (`app-authorization-ims-db2-mq`, `app-transaction-type-db2`, `app-vsam-mq`).
Security: RACF plus the USRSEC file (CSUSR01Y copybook); seeded users ADMIN001 and USER0001.

## Target architecture (defaults, confirm during planning)

- **Java 21 LTS, Spring Boot 3, Maven.** Single deployable monolith; no premature microservices.
- **Online transactions:** one Spring MVC controller + service per CICS transaction.
  Server-rendered web UI (Thymeleaf) mirroring each BMS screen, preserving field-level
  validation, error messages, and function-key semantics. REST endpoints layered under
  the screens for future service enablement.
- **Data:** VSAM KSDS to PostgreSQL via JPA. KSDS primary keys become table keys;
  alternate indexes become secondary indexes. Db2 tables (optional module) also map to
  PostgreSQL.
- **Batch:** each JCL job becomes a Spring Batch job, invocable individually and as the
  documented nightly chain (POSTTRAN, INTCALC, TRANBKP, COMBTRAN, CREASTMT, TRANIDX).
  Scheduler definitions map to cron schedules. MVSWAIT/COBSWAIT becomes a standard wait
  step; COBDATFT date conversion becomes a shared Java utility.
- **Record codecs:** copybooks become Java record classes plus a fixed-length/EBCDIC
  codec (supporting COMP, COMP-3, zoned decimal, REDEFINES, OCCURS) used to load
  PostgreSQL from `app/data/` sample files.
- **Security:** Spring Security backed by the migrated user store; preserve current
  seeded users, admin vs. regular roles, and session behavior.
- **Optional modules (late milestone, toggleable):** MQ request/response (CDRD, CDRA,
  CP00) to Amazon MQ or embedded ActiveMQ; IMS segments to relational tables.

## Non-goals

- No z/OS emulation and no byte-level 3270 protocol parity. Parity is at the
  business-function level.
- No performance/cutover engineering beyond a smoke test; that is a follow-up.
- No changes to the original COBOL sources; they remain the reference implementation
  and read-only ground truth.

## Milestones

### M0: Scaffold and readiness
Build system, CI checks, docker-compose (PostgreSQL), test harness, coding standards in
AGENTS.md, EBCDIC record codec utility, data loader skeleton, and a dev script that
starts the app + database from a clean clone. The QA scriptability requirement below
must be satisfied here.

### M1: Data foundation
Generate Java record classes from the 62 copybooks. Load all 12 sample datasets into
PostgreSQL. Round-trip parity tests: export loaded data back through the codec and
compare byte-for-byte against the source files (CBEXPORT/CBIMPORT behavior parity).

### M2: Signon and navigation
CC00 signon (Spring Security, seeded users), CM00 main menu, CA00 admin menu.

### M3: Account and card services
CAVW account view, CAUP account update, CCLI card list, CCDL card view, CCUP card
update. These are the largest programs; port business rules paragraph-by-paragraph and
unit test each rule cluster.

### M4: Transactions and billing
CT00 transaction list, CT01 view, CT02 add, CR00 reports, CB00 bill payment.

### M5: Administration
CU00 list, CU01 add, CU02 update, CU03 delete users.

### M6: Batch pipeline
Port the batch chain to Spring Batch: POSTTRAN, INTCALC, COMBTRAN, TRANBKP, CREASTMT,
TRANIDX, plus file-load jobs (ACCTFILE, CARDFILE, CUSTFILE, XREFFILE, TRANCATG,
TRANTYPE, DISCGRP, TCATBALF, DUSRSECJ). Validate each job by running it against seeded
sample data and comparing outputs/reports against expected results committed as
fixtures.

### M7: Optional modules (toggleable scope)
DB2 transaction type management (CTTU, CTLI) to JDBC/JPA; pending authorizations
(CPVS, CPVD, CP00) with IMS data mapped to tables; MQ inquiries (CDRD, CDRA) over a
local broker. Include only if explicitly kept in scope at planning time.

### M8: Hardening and cutover package
End-to-end regression suite across all user flows, performance smoke test, Dockerfile,
runbook, and a rewritten README for the Java application.

## Validation strategy

- **Characterize before porting:** for each COBOL program, extract business rules,
  field validations, and error messages into a test plan before writing the Java port.
- **Golden data:** EBCDIC/ASCII datasets under `app/data/` are the ground truth for
  both the data layer and batch outputs.
- **Per-transaction tests:** every online transaction gets unit tests for its service
  logic and a UI-level test for its screen flow.
- **Batch comparison:** every ported job runs on seeded data; outputs are diffed
  against committed expected outputs.
- **User-facing QA (must be scriptable):** a single dev script starts the app and
  database, seeds data, and resets state, so the mission's QA phase can simulate real
  user flows: signon, account view/update, card list/update, add transaction, bill
  payment, reports, admin user management.

## Acceptance criteria

1. All core online transactions are reachable through the web UI with behavior
   equivalent to the COBOL reference, including validation and error messages.
2. The core batch chain runs end-to-end on sample data with outputs matching expected
   fixtures.
3. Unit and UI test suites pass from a clean clone with one command.
4. The application starts with one documented command (docker compose + bootRun or
   equivalent) and is loadable from the provided sample data.

## Notes for the orchestrator

- Treat the COBOL sources as read-only ground truth. Never modify `app/cbl`, `app/cpy`,
  `app/jcl`, `app/bms`, or `app/data`.
- Build the new Java application in a separate top-level directory (e.g. `java/`) so the
  mainframe tree stays untouched.
- The largest programs (COACTUPC, COCRDUPC, COCRDLIC, COCRDSLC, CBTRN02C) each warrant
  their own feature breakdown; do not bundle them.
- Confirm architecture defaults (Spring Boot vs. alternatives, PostgreSQL vs. another
  database, Thymeleaf vs. SPA) with the user before generating the detailed plan.
