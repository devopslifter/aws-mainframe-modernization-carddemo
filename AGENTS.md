# AGENTS.md

CardDemo: a mainframe credit card management application (COBOL, CICS, VSAM KSDS, JCL,
plus optional Db2/IMS/MQ modules), used as the read-only reference system for a Java
migration mission. The mission brief lives in `mission.md` at the repo root.

## Ground rules

- Everything under `app/` is read-only ground truth (COBOL, copybooks, JCL, BMS,
  assembler, sample data). Never modify, move, or delete it.
- `app/data/EBCDIC` and `app/data/ASCII` hold the golden sample datasets used for
  migration validation. Treat them as immutable fixtures.
- All new Java application code goes under `java/` (created by the migration mission).
- `mission.md` is the mission brief: architecture decisions, milestones, validation
  strategy, and acceptance criteria. Keep it in sync if scope changes.

## Repository map

| Path | Contents |
| :-- | :-- |
| `app/cbl` | 44 COBOL programs (online CICS transactions and batch) |
| `app/cpy`, `app/cpy-bms` | 62 copybooks (record layouts; includes COMP, COMP-3, REDEFINES, OCCURS) |
| `app/bms` | 21 BMS mapsets (screen definitions) |
| `app/jcl`, `app/proc` | 46 batch jobs and cataloged procedures |
| `app/asm`, `app/maclib` | Assembler utilities: MVSWAIT, COBDATFT |
| `app/data` | Golden sample data, EBCDIC and ASCII (12 datasets) |
| `app/csd`, `app/catlg`, `app/ctl`, `app/scheduler` | CICS resource definitions, catalog, control cards, Control-M and CA7 scheduler definitions |
| `app/app-authorization-ims-db2-mq`, `app/app-transaction-type-db2`, `app/app-vsam-mq` | Optional modules (IMS/DB2/MQ authorizations, DB2 transaction types, VSAM/MQ) |
| `samples/`, `scripts/`, `diagrams/` | Compile samples, helper scripts, application flow diagrams |
| `README.md` | Authoritative inventory: transaction IDs, batch job chain, dataset/copybook mapping tables |

## Working notes

- Naming: `CO*` programs are online CICS transactions; `CB*` programs are batch.
  Transaction IDs, their programs, and their BMS maps are listed in the README
  inventory tables.
- The largest programs (COACTUPC, COCRDUPC, COCRDLIC, COCRDSLC, CBTRN02C) are
  50-180 KB each. Decompose their analysis and porting; do not treat them as single
  units.
- Seeded users from the README: ADMIN001 and USER0001 (password `PASSWORD`).
- The COBOL tree has no build or test system by design; do not add one. Validation
  lives in `java/`: one-command startup, golden-data parity tests against `app/data`,
  and batch output fixtures. See `mission.md` for specifics.
- Mainframe sources use fixed-format 80-column records; account for that when
  reading or diffing them.
