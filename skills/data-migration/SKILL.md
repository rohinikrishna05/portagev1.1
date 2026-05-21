---
name: data-migration
description: Plan and (with confirmation) execute data migrations from AWS data services to GCP equivalents — RDS to Cloud SQL or AlloyDB, ElastiCache to Memorystore, S3 to GCS, MSK to GCP Kafka or Confluent Cloud — and the secrets/config moves that go with them. Produces per-data-system runbooks, replication plans, validation gates, and cutover scripts. Use after storage-translation, when "plan our RDS migration", "migrate Redis to Memorystore", or "data layer plan for the GKE move".
---

# Data Migration

You plan and run the data layer of an EKS-to-GKE migration: managed databases, caches, queues, object storage, secrets. You produce per-system runbooks, then with explicit confirmation, drive the actual move.

## Purpose

Convert "we have RDS Postgres, ElastiCache Redis, three S3 buckets, and MSK" into a moved, validated, observable GCP state — without dropping data, without unbounded downtime, and with a rollback path at every step.

## When to use this skill

- Phase 4 of a Portage migration.
- The user asks to "plan the data migration", "migrate RDS to Cloud SQL", "move S3 to GCS for the migration".

## Prerequisites

- `01-discovery/inventory.json` data dependencies section.
- `02-assessment/readiness-report.md` with effort estimate per system.
- `03-landing-zone/plan.md` with the `data-prod` project provisioned.
- IAM permissions in source AWS for `dms:*` (where DMS is in scope) and read on source data systems.
- IAM permissions in GCP for the data services in scope (`cloudsql.admin`, `redis.admin`, etc.).

## Procedure

### Step 1 — Per-system plan

For each data system in `inventory.json`, produce a plan document. Use one of the templates below.

#### Template — RDS for PostgreSQL → Cloud SQL for PostgreSQL (homogeneous)

```
System:           prod-payments-db
Source:           RDS Postgres 
Target:           Cloud SQL Postgres
RPO target:       < 30s
RTO target:       < 15 min
Strategy:         Database Migration Service (continuous CDC), then switch primary
```

**Procedure (grounded in [DMS — configure Postgres source](https://docs.cloud.google.com/database-migration/docs/postgres/configure-source-database) and [DMS — Postgres known limitations](https://docs.cloud.google.com/database-migration/docs/postgres/known-limitations)):**

1. **Source RDS parameter group** — create a new parameter group, attach to the instance, restart:
   - `shared_preload_libraries` includes `pglogical`.
   - `rds.logical_replication = 1` (RDS-managed; this enables WAL at the `logical` level — do NOT also set `wal_level = logical` directly on RDS).
   - `wal_sender_timeout = 0`.
   - `max_replication_slots` ≥ (DBs being migrated × concurrent migration jobs) + existing usage. Default is 10.
   - `max_wal_senders` ≥ `max_replication_slots` + existing senders.
   - `max_worker_processes` ≥ DBs being migrated + existing usage.

2. **Per-database setup on every database** (except `template0`, `template1`, `rdsadmin`):

   ```sql
   CREATE EXTENSION IF NOT EXISTS pglogical;
   ```

3. **User privileges** — on each database, for every schema other than `information_schema` and any beginning with `pg_`:

   ```sql
   GRANT USAGE  ON SCHEMA <schema>          TO <migration_user>;
   GRANT USAGE  ON SCHEMA pglogical         TO PUBLIC;
   GRANT SELECT ON ALL TABLES    IN SCHEMA pglogical  TO <migration_user>;
   GRANT SELECT ON ALL TABLES    IN SCHEMA <schema>   TO <migration_user>;
   GRANT SELECT ON ALL SEQUENCES IN SCHEMA <schema>   TO <migration_user>;
   ```

   For RDS specifically (RDS restricts SUPERUSER, so `ALTER USER ... WITH REPLICATION` does not apply):

   ```sql
   GRANT rds_replication TO <migration_user>;
   ```

4. **Network reachability** — the source must be reachable by DMS via IP allowlist (public IP), VPC peering, or reverse SSH tunnel. For a private-IP target, enable the **Service Networking API** in the project; the operator needs `servicenetworking.services.addPeering` and `compute.networkAdmin`.

5. **Provision Cloud SQL target via DMS migration-job wizard.** New-instance flow:
   - Type: `New instance`. The destination's `postgres` admin password is set in the wizard.
   - Edition: Cloud SQL for PostgreSQL Enterprise (or Enterprise Plus).
   - Region/zone, optionally `Multiple zones (Highly available)`.
   - Connectivity: For source with public IP/DNS use IP Allowlist(Get SSL cert using curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem). For source with private IP use Private IP (VPC peering — note: when DMS creates the destination, **only VPC peering is supported for private IP**; Private Service Connect requires migrating to a pre-existing instance).
   - Storage ≥ source DB size.
   - Encryption: Google-managed default, or CMEK with a key resource name like `projects/<p>/locations/<l>/keyRings/<kr>/cryptoKeys/<k>`.
   - Match source database flags where supported.

6. **Tables without primary keys**: only the initial snapshot and `INSERT` statements during CDC are migrated. `UPDATE` and `DELETE` must be migrated manually. Add a PK or follow the docs workaround.

7. **Initial snapshot + continuous CDC.** Validate row counts per table; sample row hashes match. Monitor replication lag — target lag < 30 s for 5 minutes before cutover.

8. **Cutover window:**
   a. Stop writers on EKS app side (drain).
   b. Wait for replication lag = 0.
   c. Promote target. Disable replication.
   d. Repoint app config to Cloud SQL connection string (this is a `traffic-cutover` step).
   e. **Re-enable point-in-time recovery and re-apply any custom backup settings** — DMS resets backup settings during promotion.
   f. Smoke tests on target.

9. **Soak.** Keep RDS read-only for 14 days as rollback.

10. **Decommission RDS.**

**Postgres-specific limitations to surface up-front (from the DMS known-limitations page):**

- `pglogical` does not replicate generated columns for PostgreSQL 12+.
- DDL changes are not replicated via standard SQL; only via `pglogical.replicate_ddl_command`. New tables must be added with `pglogical.replication_set_add_table`.
- Materialized views: only the schema is migrated; run `REFRESH MATERIALIZED VIEW <name>` post-cutover.
- `SEQUENCE` `last_value` may differ on the destination — verify and adjust if app logic depends on it.
- `UNLOGGED` and `TEMPORARY` tables aren't (and can't be) replicated.
- Large Object data type isn't supported.
- Only Cloud-SQL-supported extensions and procedural languages migrate; unsupported ones are silently skipped during the test/start of the job.
- `pg_cron` extension and its `cron` settings aren't migrated — re-install on destination after cutover.
- Cannot migrate from a read replica in recovery mode.
- Doesn't support RDS sources where the AWS SCT extension pack is applied.
- C-language UDFs can't be migrated (except via supported extensions).
- Databases added after the job starts aren't migrated.
- Cannot select specific tables/schemas — DMS migrates everything except `information_schema` and `pg_*` catalogs.
- Users and roles aren't migrated — recreate on the destination.
- Customized tablespaces collapse to `pg_default` on the destination.
- Quotas: 2,000 connection profiles, 1,000 migration jobs.

**Validation gates:**
- Replication lag < 30 s for 5 minutes.
- Row count parity per critical table.
- Random row hash sample matches.
- No errors in target Cloud SQL log for 30 minutes pre-cutover.

#### Template — RDS for MySQL (or Aurora MySQL) → Cloud SQL for MySQL

Grounded in [DMS — configure MySQL source](https://docs.cloud.google.com/database-migration/docs/mysql/configure-source-database) and [DMS — MySQL known limitations](https://docs.cloud.google.com/database-migration/docs/mysql/known-limitations).

**Supported source versions:** Amazon RDS 5.6, 5.7, 8.0, 8.4. (8.0 minor versions: 8.0.18, .26–.28, .30–.37, .39–.43.)

**Source flags / parameter group:**
- `server-id` — set to a value of 1 or larger.
- `GTID_MODE` — `ON` or `OFF`. **`ON_PERMISSIVE` is not supported** by DMS — common RDS default; check yours. Must be `ON` if migrating to a destination that has read replicas, or if using a manual dump.
- Binary logging enabled, **`ROW`-format**. Verbatim from Google: "You must set the binary log format to ROW. Configuring the binary log to any other format, such as STATEMENT or MIXED, might cause the replication to fail."
- Binlog retention long enough for replication; recommended 168 hours (7 days, the RDS maximum).

**RDS-specific binlog retention command (verbatim):**

```
call mysql.rds_set_configuration('binlog retention hours', 168);
```

For Aurora, the same call applies.

**Migration user:**
- Configure with `host = '%'`.
- **Password ≤ 32 characters** — MySQL replication limitation (MySQL Bug #43439). Easy to miss; longer passwords silently break.
- For MySQL 8.0+, the migration account must **not** have `BACKUP_ADMIN`.

**Privileges by migration type (verbatim):**

| Type | Privileges |
|---|---|
| Continuous + managed dump | `REPLICATION SLAVE`, `EXECUTE`, `SELECT`, `SHOW VIEW`, `REPLICATION CLIENT`, `RELOAD`, `TRIGGER`; plus `LOCK TABLES` for RDS/Aurora. |
| Continuous + manual dump | `REPLICATION SLAVE`, `EXECUTE`. |
| One-time + managed dump | `SELECT`, `SHOW VIEW`, `TRIGGER`; plus `LOCK TABLES` for RDS/Aurora; plus `RELOAD` if `GTID_MODE = ON`. |
| One-time + manual dump | (none) |

**Storage engine:** all tables (except system DBs) must use **InnoDB**. MyISAM may cause data inconsistency.

**DDL during migration:** stop all DDL writes during the full-dump phase (script provided in the docs). DDL may resume after CDC begins.

**Aurora-specific:** cannot migrate from an Aurora *read replica* — binary logs aren't retrievable from the replica.

**RDS quirk Google calls out:** "If the source is Amazon RDS MySQL, Amazon Aurora MySQL, or a source that doesn't grant SUPERUSER privileges, then additional steps are required for successful migration, including a brief write downtime on the source." Plan for that downtime in the cutover window.

**Target (Cloud SQL for MySQL, new instance via DMS):**
- Edition: Cloud SQL for MySQL Enterprise (or Enterprise Plus).
- Storage type SSD or HDD; storage capacity ≥ source size (cannot shrink later).
- For cross-version 8.0 → 8.4 migrations: destination must have `local_infile = ON`.
- Match other database flags to the source where supported.
- Connectivity: private IP via VPC peering (Private Service Connect not available for new-instance flow), or public IP with authorized networks.

**MySQL-specific limitations to surface up-front:**

- DMS is not compatible with MariaDB.
- Migrating to MySQL 5.6 or 8.4 with a Percona XtraBackup physical file is not supported.
- Data dump parallelism is only available for destinations on MySQL 5.7 or 8.
- The MySQL system database isn't migrated; user roles aren't included — recreate on destination.
- System schemas `mysql`, `performance_schema`, `information_schema`, `sys` are always excluded; objects referencing them fail with `ERROR 1109 (42S02): Unknown table in <schema name>`.
- For continuous migration with your own dump, do **not** use `mysqldump` from MySQL 5.7.36 (bug #105761).
- Only InnoDB is supported on Cloud SQL.
- Data-dump parallelism briefly locks the source: 100 tables ≈ 1 s, 10K ≈ 9 s, 50K ≈ 49 s. Use a read replica if locking is unacceptable.
- Quotas: 2,000 connection profiles, 1,000 migration jobs.

#### Template — RDS / Aurora → AlloyDB (engine change for higher perf)

Surface as escalation. AlloyDB is engine-compatible with Postgres but has different performance characteristics, costs, and feature surface. Engine change is a project of its own.

#### Template — ElastiCache Redis → Memorystore for Redis Cluster

```
System:           prod-session-cache
Source:           ElastiCache Redis Cluster Mode 7.x, 3 shards × 2 replicas
Target:           Memorystore for Redis Cluster, 3 shards
Strategy:         Application-driven (cache, can rebuild) OR replica seed
```

If the cache is **rebuildable** (sessions, ephemeral data): provision target empty, switch app config at cutover, accept warm-up cost.

If the cache **must persist** (counters, rate-limit state, durable Redis features): use `RIOT-X` or similar to do an online seed + tail-based replication, or accept a brief cutover with `--rdb` snapshot import.

#### Template — S3 → GCS

```
System:           shop-orders bucket
Source:           s3://shop-orders, 8 TiB, ~12M objects, lifecycle policies
Target:           gs://shop-orders, multi-regional or regional based on access pattern
Strategy:         Storage Transfer Service (incremental)
```

**Procedure:**
1. Create GCS bucket (regional, same region as primary GKE cluster typically; multi-regional only when access is multi-region). Set encryption, lifecycle rules to mirror S3.
2. Provision Storage Transfer Service job. Initial sync.
3. Continuous incremental sync (STS supports event-driven from S3).
4. Validate object counts and a hash sample.
5. At app cutover: app config repoints to GCS bucket. Keep S3 as read-only during soak.
6. After soak, retire STS job, retire S3 bucket.

#### Template — MSK Kafka → GCP-hosted Kafka (Confluent Cloud or self-managed)

GCP has no first-party managed Kafka GA at time of writing. Confirm current state before committing. Strategies:

- **Confluent Cloud on GCP**: provision cluster, mirror topics with Confluent Replicator or MirrorMaker 2 from MSK, switch consumer groups at cutover.
- **Self-managed Kafka on GKE**: deploy via Strimzi operator on GKE, mirror with MM2.

#### Template — DynamoDB → Bigtable / Spanner / Firestore (heterogeneous)

This is a re-platform, not a migration. `data-migration` produces only a scoping note and hands back to the user / a separate project.

#### Template — Secrets Manager → Secret Manager

```
For each Secret in AWS Secrets Manager referenced by workloads in scope:
  1. Create equivalent Google Cloud Secret with same name (or namespaced).
  2. Replicate value (use a one-time script with read-only AWS creds).
  3. Update KSA bindings (identity-translation already covers these).
  4. At workload cutover: app reads from Secret Manager via the External Secrets Operator
     pointing at GCP, or via SDK.
  5. Keep AWS Secret for rollback period.
```

For values that are credentials to the *source* AWS environment, retire after cutover; do not migrate them long-term.

### Step 2 — Co-existence connectivity

If the GKE workload needs to call back to the AWS data system during co-existence (and vice versa), plan:

- VPN or Cloud Interconnect between AWS VPC and GCP VPC.
- Allow-list updates to source data systems' security groups (allow GCP CIDRs).
- TLS / network policy parity.

Bandwidth budget: project peak data movement at the planned co-existence window. Surface costs.

### Step 3 — Per-system runbook with explicit gates

For each system, render a runbook (use `templates/runbook-template.md`) with:

- Pre-flight checklist.
- Cutover steps with expected commands and outputs.
- Validation gate after each step.
- Rollback procedure for each step (linking to `rollback-playbook`).
- Decommission steps and timing.

### Step 4 — Drive the migration (with explicit confirmation)

For each system:

1. Surface the full runbook to the user. Ask explicit confirmation: "Begin replication for `prod-payments-db`? Estimated initial sync: 4–6 hours. Cost: $X."
2. On confirmation, execute step-by-step. After each step, run the validation gate. If a gate fails, stop, do not proceed.
3. Log every command executed and its output to `10-data-migration/<system>/execution.log`.

The agent never destructively modifies the source. The agent never bypasses a failed gate.

### Step 5 — Output

```
10-data-migration/
├── plan.md                       # Index of all systems
├── <system-1>/
│   ├── runbook.md
│   ├── pre-flight-checks.md
│   ├── execution.log              # Filled in during execution
│   ├── validation-gates.md
│   └── rollback.md                # Links into rollback-playbook
├── <system-2>/
└── escalations.md
```

## Decision points

| Decision                                  | Default                                          | When to deviate                              |
|-------------------------------------------|--------------------------------------------------|----------------------------------------------|
| Postgres / MySQL replication tool          | DMS                                              | pg_logical native if DMS network constraints |
| Redis seed strategy                        | Rebuildable → empty target; durable → snapshot import | Online tail-based replication for true zero-downtime |
| S3 → GCS engine                            | Storage Transfer Service                         | gsutil/`gcloud storage` for one-shot, small datasets |
| MSK target                                 | Confluent Cloud on GCP                           | Self-managed Kafka if cost or operator is fluent |
| DynamoDB target                            | Out of scope; re-platform project                | (No default) |
| Soak duration on source                    | 14 days read-only                                | 7 days for non-prod, longer for tier-0 |

## Outputs / Deliverables

```
10-data-migration/
├── plan.md
├── <one-folder-per-system>/
└── escalations.md
```

## Validation

For each system, the cutover gate criteria must be defined and met before declaring complete:

- Postgres / MySQL: replication lag < target SLA for ≥ 5 min, row counts match for top 10 tables, sample row hashes match.
- Redis: target writeable + readable, app smoke tests pass on cached paths.
- S3 → GCS: object count matches, sample hashes match, lifecycle rules verified active on GCS.
- Kafka: consumer offset translation verified, MM2 lag ≈ 0, end-to-end produce → consume across both sides during co-existence works.

Post-cutover, source remains read-only for the configured soak window. Rollback during soak is a documented, exercised path.

## Escalation triggers

- Heterogeneous data store moves (DynamoDB, Aurora→AlloyDB, Neptune, Redshift→BigQuery). Surface as scoping requirements.
- Datasets where the time-to-replicate exceeds the user's available window.
- Encryption key migrations that cannot reuse a CMEK approach (HSM-only key material). Surface for KMS planning.
- Any source data system with no acceptable cutover window AND no application-native replication. The migration of that system stops until the user agrees to one of: (a) accept downtime, (b) rebuild downstream with new write-path, (c) defer this system out of the migration.

## Common pitfalls

- **Forgetting parameter group parity.** RDS parameter groups (e.g., `max_connections`, `shared_preload_libraries`) translate to Cloud SQL flags. Mismatch causes subtle prod issues.
- **Cloud SQL connection methods.** Public IP is convenient but private IP via VPC peering or PSC is the standard for prod. Use the [Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/postgres/connect-auth-proxy) sidecar or Workload Identity-aware connector.
- **S3 ACLs / signed URLs.** S3 signed URLs do not work on GCS. Translate to GCS signed URLs (different signing algorithm) at the application layer before cutover.
- **Cache stampede at cutover.** Empty Memorystore + cold app = thundering herd. Plan a warm-up via shadow traffic or a back-end pre-fill.
- **Kafka offset translation.** Source offsets do not equal target offsets. MM2 maintains a translation table; consumer groups must use it.
- **Egress during co-existence routinely runs 5–10× the unbudgeted estimate.** Model it explicitly per-workload at assessment time and monitor near-real-time during execution. See [LFF-01](../../reference/lessons-from-the-field.md#lff-01--egress-during-co-existence-routinely-runs-510-the-unbudgeted-estimate).
- **Cross-cloud bulk transfer over open internet bottlenecks at default Linux TCP buffers.** Tune `net.ipv4.tcp_{r,w}mem` before measuring throughput; otherwise you'll mis-time the cutover window. See [LFF-11](../../reference/lessons-from-the-field.md#lff-11--zfs-send-over-wan-bottlenecks-at-30-mbs-on-default-linux-tcp-buffers).
- **Cloud SQL hides behind Google-controlled VPC peering** and routes do *not* propagate through a second peering hop. Co-locate or use Private Service Connect; do not plan a transit-VPC hop. See [LFF-12](../../reference/lessons-from-the-field.md#lff-12--cloud-sql-hides-behind-google-controlled-vpc-peering-blocking-second-hop-route-propagation).
- **DMS new-instance flow only supports VPC peering for private IP.** If PSC is required, pre-create the Cloud SQL instance and migrate to it. See [LFF-13](../../reference/lessons-from-the-field.md#lff-13--cloud-sql-dms-new-instance-flow-only-supports-vpc-peering-for-private-ip).
- **Postgres replication slots can vanish silently** on managed-DB failover, causing CDC consumers to lose rows. Monitor slot existence + lag continuously during the cutover window. See [LFF-15](../../reference/lessons-from-the-field.md#lff-15--postgres-replication-slots-can-be-dropped-silently-on-managed-db-failover).

## References

- **Canonical sources**: [reference/sources.md](../../reference/sources.md).
- [Database Migration Service overview](https://cloud.google.com/database-migration/docs).
- [DMS — configure Postgres source (RDS)](https://docs.cloud.google.com/database-migration/docs/postgres/configure-source-database).
- [DMS — configure MySQL source (RDS)](https://docs.cloud.google.com/database-migration/docs/mysql/configure-source-database).
- [DMS — Postgres known limitations](https://docs.cloud.google.com/database-migration/docs/postgres/known-limitations).
- [DMS — MySQL known limitations](https://docs.cloud.google.com/database-migration/docs/mysql/known-limitations).
- [Storage Transfer Service](https://cloud.google.com/storage-transfer/docs).
- [Migrate from Amazon RDS / Aurora MySQL to Cloud SQL for MySQL (Architecture Center)](https://docs.cloud.google.com/architecture/migrate-aws-rds-to-sql-mysql).
- [storage-translation](../storage-translation/SKILL.md) — provides StorageClass, in-cluster volume strategies.
- [traffic-cutover](../traffic-cutover/SKILL.md) — sequences workload + data cutovers.
- [rollback-playbook](../rollback-playbook/SKILL.md) — gates failing during data migration cutover.
- [docs/glossary.md](../../docs/glossary.md) — service map.
