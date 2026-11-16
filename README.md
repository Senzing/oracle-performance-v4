<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/senzing-logo-dark-theme.png">
    <img src="assets/senzing-logo-light-theme.png" alt="Senzing" width="240">
  </picture>
</p>

<h1 align="center">Oracle performance</h1>

<img src="assets/ember-rule.svg" alt="" width="100%" height="4">

Performance tuning for running **SENZING® SDK v4** on **Oracle Database 26ai** under high-throughput load —
companion to
[performance-general-v4](https://github.com/Senzing/performance-general-v4/blob/main/README.md). Apply
everything below **after** the standard Senzing SDK v4 schema and **before** loading.

This page is about getting the most out of Oracle with the Senzing SDK. It is not a comparison with other
databases.

Each section is written to stand on its own: it names the Senzing tables and Oracle settings it concerns,
explains the mechanism and gives **Do this**, **Avoid** and **How to check** (a query or command you can run
on your own system). Setting values are recommendations; measure the effect on your own hardware and data
with the checks given.

> [!NOTE]
> **Work in progress.** This page is being written as our Oracle v4 testing proceeds. Sections are added once a
> finding is confirmed from an empty datastore; provisional findings are called out as such.

The hardware, operating systems and software we tested on are in [TEST_ENVIRONMENT.md](TEST_ENVIRONMENT.md).

⭐ marks a lever that made a large difference in our own v4 testing.

## At a glance — the key levers

_Added as levers are measured._

- ⭐ **No histograms on the Senzing schema** — `METHOD_OPT = 'FOR ALL COLUMNS SIZE 1'`. A histogram on a generated
  ID column can switch the hottest lookup to a full table scan whose cost grows with the table.
  [Details](#-no-histograms-on-the-senzing-schema)
- ⭐ **Load with representative statistics, locked.** Statistics from a sizeable sample load (or set by hand), then
  `LOCK_SCHEMA_STATS`; they need only say the tables are large. [Details](#-load-with-representative-statistics-locked)
- ⭐ **Spread the network interrupts across all sockets.** [Details](#spread-the-network-interrupts-across-all-processor-sockets)
- **Keep the maintenance-window resource plan out of a load.** [Details](#keep-the-maintenance-window-resource-plan-out-of-a-load)

---

# Senzing settings for Oracle

## Entity lock mode — LEASE

Senzing's `ADVISORY` entity lock mode is implemented for PostgreSQL and SQL Server only. On Oracle the engine
uses `LEASE`.

**Do this:** set `ENTITY_LOCK_MODE` to `LEASE` explicitly in the engine configuration of **every** process that
connects to the datastore, so that the lock mode recorded in the datastore is unambiguous. All processes on
one datastore must use the same lock mode.

## `RES_ENT.FEATURES` store _(v4.4+)_

The engine's feature store on `RES_ENT` collapses the feature reads per record. The standard Oracle schema
does not create the column.

**Do this:** add it to the schema the engine connects to, before the first engine initialization (the engine
detects the column when it starts):

```sql
ALTER TABLE RES_ENT ADD FEATURES CLOB;
```

**How to check:** after loading a few records, this returns rows:

```sql
SELECT RES_ENT_ID FROM RES_ENT WHERE FEATURES IS NOT NULL FETCH FIRST 5 ROWS ONLY;
```

---

# Optimizer statistics

Every Senzing SDK statement is a key lookup or key-driven write with bind variables, and the tables start empty
and grow by orders of magnitude during a load. Oracle's defaults are tuned for the opposite case. Apply this
section to **every** database the engine connects to — every node of a multi-node datastore.

## ⭐ No histograms on the Senzing schema

**Tables:** all, most visibly `RES_FEAT_EKEY` (`LIB_FEAT_ID`). The engine's candidate lookup,
`SELECT ... FROM RES_FEAT_EKEY WHERE LIB_FEAT_ID IN (:0 ... :31) AND RES_ENT_ID != :32`, is the hottest
statement during a load.

**Mechanism.** Oracle's automatic statistics (`METHOD_OPT` default `FOR ALL COLUMNS SIZE AUTO`) build a histogram
on `LIB_FEAT_ID` once some features are shared by many entities. When the cursor is hard parsed, Oracle peeks at
the bind values; if they fall on histogram-"popular" ids the estimate jumps and the plan becomes a
`TABLE ACCESS FULL`. That plan is then shared by every execution, and its cost grows with the table, so
throughput falls continuously as the load proceeds. In our testing, Oracle did not move the cursor off that plan
on its own while the load ran. The ids are generated keys, so the "popularity" carries no useful
information; after the datastore is purged they even refer to different features.

With identical statistics otherwise, the same statement with 32 non-popular ids produces
`INLIST ITERATOR` → `INDEX RANGE SCAN RES_FEAT_EKEY_PK`.

**Do this** (as a DBA, before loading; repeat for every node):

```sql
EXEC DBMS_STATS.SET_SCHEMA_PREFS('<SENZING_SCHEMA>', 'METHOD_OPT', 'FOR ALL COLUMNS SIZE 1');

-- remove histograms already gathered
BEGIN
  FOR r IN (SELECT table_name, column_name FROM dba_tab_col_statistics
            WHERE owner = '<SENZING_SCHEMA>' AND histogram <> 'NONE') LOOP
    DBMS_STATS.DELETE_COLUMN_STATS('<SENZING_SCHEMA>', r.table_name, r.column_name,
                                   col_stat_type => 'HISTOGRAM', no_invalidate => FALSE);
  END LOOP;
END;
/
```

**Avoid:** fixing a single bad plan by flushing it from the shared pool. The next hard parse can peek
popular ids again.

**How to check:**

```sql
-- no histograms remain
SELECT table_name, column_name, histogram FROM dba_tab_col_statistics
WHERE owner = '<SENZING_SCHEMA>' AND histogram <> 'NONE';

-- no Senzing statement is scanning a whole table while loading
SELECT s.sql_id, p.object_name, s.executions,
       ROUND(s.buffer_gets / NULLIF(s.executions, 0)) AS gets_per_exec
FROM   v$sql s JOIN v$sql_plan p ON p.sql_id = s.sql_id AND p.child_number = s.child_number
WHERE  s.parsing_schema_name = '<SENZING_SCHEMA>'
AND    p.operation = 'TABLE ACCESS' AND p.options = 'FULL' AND p.object_owner = '<SENZING_SCHEMA>'
ORDER  BY s.cpu_time DESC;
```

A full scan whose `gets_per_exec` rises over time is this problem. Both queries return no rows when the setting is
in place.

## ⭐ Load with representative statistics, locked

**Tables:** all; seen first on `RES_REL_EKEY` (`DELETE ... WHERE (RES_ENT_ID, REL_ENT_ID, RES_REL_ID) IN (...)` and
`SELECT ... WHERE RES_ENT_ID IN (...)`).

**Mechanism.** A plan is costed against the statistics present when it is parsed. Statistics gathered on an empty or
nearly empty table — after a fresh schema or a purge, or by a maintenance-window job early in a load — make a full
scan look cheapest. The plan stays until the statistics are regathered and the cursor invalidated, while the table
grows far past the size it was costed for.

The statistics do not need to be accurate. Every Senzing SDK statement is written for the Senzing indexes; Oracle only
needs statistics that say the tables are large, so that an index access is always the cheaper plan. Statistics from
any sizeable load of similar data are enough, and they stay valid as the datastore grows past them.

**Do this** (as a DBA, with no Senzing processes connected; repeat for every node):

1. Load a representative sample — millions of records of your own data — and gather once, without histograms:

   ```sql
   EXEC DBMS_STATS.GATHER_SCHEMA_STATS('<SENZING_SCHEMA>', method_opt => 'FOR ALL COLUMNS SIZE 1', cascade => TRUE, no_invalidate => FALSE);
   ```

2. Save them, so they can be re-imported into a rebuilt schema
   (`DBMS_STATS.CREATE_STAT_TABLE`, `EXPORT_SCHEMA_STATS`, later `IMPORT_SCHEMA_STATS`).
3. Lock them, so neither the automatic statistics job nor a manual gather replaces them:

   ```sql
   EXEC DBMS_STATS.LOCK_SCHEMA_STATS('<SENZING_SCHEMA>');
   ```

Purging the datastore with the SDK truncates the tables and leaves the statistics in place, so a purged datastore is
ready to load again. Instead of a sample load you can set the figures directly with `DBMS_STATS.SET_TABLE_STATS` and
`SET_INDEX_STATS` before locking.

**Avoid:** gathering statistics on a schema that is empty or just purged, and leaving the schema unlocked during a load
that spans a maintenance window.

**How to check:** the full-scan query in the previous section returns no rows, and

```sql
SELECT table_name, num_rows, stattype_locked FROM dba_tab_statistics
WHERE owner = '<SENZING_SCHEMA>' AND object_type = 'TABLE' ORDER BY num_rows DESC;
```

shows non-trivial `NUM_ROWS` and `STATTYPE_LOCKED = 'ALL'` for the Senzing tables.

## Keep the maintenance-window resource plan out of a load

Oracle's default maintenance windows (weeknights, and all of Saturday and Sunday) activate
`DEFAULT_MAINTENANCE_PLAN`, which manages CPU and runs the automatic statistics job alongside the load. A load that
spans a window boundary changes behavior partway through, and the statistics job can gather on tables that are
still nearly empty (see above). Under the plan, sessions waiting for CPU show as `resmgr:cpu quantum`.

**Do this** for a dedicated loading datastore, in the CDB and in the PDB:

```sql
ALTER SYSTEM SET resource_manager_plan = 'FORCE:' SCOPE = BOTH;
```

`FORCE:` with no plan name keeps the scheduler windows from switching plans. Set it back to your site's plan when
the load is finished.

**How to check:** `SELECT value FROM v$parameter WHERE name = 'resource_manager_plan';` returns `FORCE:`, and
`resmgr:cpu quantum` does not appear among the top waits.

---

# Host settings for the database server

## Spread the network interrupts across all processor sockets

**Mechanism.** The Senzing SDK talks to the database in many small round trips over many connections. Each network
interface queue raises its interrupts, and processes its packets, on the CPUs its interrupt is bound to. Without
an interrupt balancer, network drivers commonly bind every queue to the CPUs of the socket (NUMA node) the card is
attached to. The database threads serving those connections then crowd onto that socket: it runs out of CPU while the
other socket idles. Oracle reports the queueing as I/O wait — `db file sequential read` averages several times the
storage's own read latency.

**Do this:** bind the interface's queue interrupts across the CPUs of **all** sockets, for example one queue per CPU
(`/proc/irq/<n>/smp_affinity_list`, as root). The setting does not survive a reboot; apply it from the host's boot
configuration. Do not assume `irqbalance` does this: it may keep a device's interrupts on the device's own node — check
the result.

**How to check:**

```sh
# per-socket CPU use: one node near 100% (with high %soft) while another is mostly idle is this problem
mpstat -N ALL 5 1
# which CPUs each queue interrupt is bound to
for i in $(awk -F: '/<interface>/ {print $1}' /proc/interrupts); do cat /proc/irq/$i/effective_affinity_list; done
```

and compare Oracle's average `db file sequential read` (`V$SYSTEM_EVENT`) with the device read latency
(`iostat -x`, `r_await`). After the change they should be close.

---

# Startup requirements — the Oracle container image with 1 GiB huge pages

These apply to the Oracle Database Enterprise Edition container image
(`container-registry.oracle.com/database/enterprise`) on a Linux host whose default huge page size is 1 GiB.
Each one stopped an instance from starting, or left it running without huge pages.

## 1. The `oracle` user must be allowed to use huge pages

The database runs as the non-root `oracle` user. `--cap-add IPC_LOCK` alone does not let it map huge pages;
the user must also be in the group named by `vm.hugetlb_shm_group`. Without it the System Global Area (SGA)
falls back to 4 KiB pages **silently** — the instance starts and `HugePages_Free` does not move.

**Do this:** add the container to that group, for example `--group-add 0` when `vm.hugetlb_shm_group = 0`.

**How to check:** `HugePages_Free` in `/proc/meminfo` on the host drops by about the SGA size after the
instance starts.

## 2. `/dev/shm` must be large enough

Part of Oracle's memory lives in `/dev/shm`. Docker's default of 64 MB fails at startup with `ORA-04030` /
`ORA-27300` (`sskgm_mga_cr`).

**Do this:** start the container with `--shm-size` sized for the instance (we use `--shm-size=32g`).

## 3. Background processes need scheduling priority

Without the capability to raise priority, the log writer and timekeeper processes report `ORA-00800 Set
Priority Failed`.

**Do this:** start the container with `--cap-add SYS_NICE`.

## 4. Keep `use_large_pages = TRUE`

With 1 GiB huge pages, `use_large_pages = ONLY` failed to start with `ORA-27123: unable to attach to shared
memory segment`. `TRUE` places most of the SGA on huge pages.

**Avoid:** `use_large_pages = ONLY` on a host with 1 GiB huge pages.

## 5. Restart with the container, not with `SHUTDOWN` / `STARTUP`

When a startup fails, the image's start script (`startDB.sh`) retries it with `SHUTDOWN ABORT` followed by
`STARTUP`. A `STARTUP` issued by hand while it retries is aborted under you:
`ORA-03113: end-of-file on communication channel`.

**Do this:** change parameters with `ALTER SYSTEM ... SCOPE=SPFILE`, then `docker restart` the container.
