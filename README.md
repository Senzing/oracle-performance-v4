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

All findings on this page were measured with **SENZING® SDK 4.5.0**. The hardware, operating systems and software
we tested on are in [TEST_ENVIRONMENT.md](TEST_ENVIRONMENT.md).

⭐ marks a lever that made a large difference in our own v4 testing.

## At a glance — the key levers

_Added as levers are measured._

- ⭐ **No histograms on the Senzing schema** — `METHOD_OPT = 'FOR ALL COLUMNS SIZE 1'`. A histogram on a generated
  ID column can switch the hottest lookup to a full table scan whose cost grows with the table.
  [Details](#-no-histograms-on-the-senzing-schema)
- ⭐ **Load with representative statistics, locked.** Statistics from a sizeable sample load (or set by hand), then
  `LOCK_SCHEMA_STATS`; they need only say the tables are large. [Details](#-load-with-representative-statistics-locked)
- ⭐ **Spread the network interrupts across all sockets.** [Details](#spread-the-network-interrupts-across-all-processor-sockets)
- ⭐ **New repositories with Chinese, Japanese or Korean data: turn transliterated keys off.** Keep the translated keys.
  [Details](#-new-repositories-with-chinese-japanese-or-korean-data-turn-transliterated-keys-off-v45)
- **Store short text columns as `VARCHAR2(32767)` with `MAX_STRING_SIZE = EXTENDED`, not `CLOB`.**
  `LIB_FEAT.FELEM_VALUES` as a CLOB is stored as UTF-16 in an `AL32UTF8` database, which makes the largest feature
  table substantially bigger. `VARCHAR2(4000)` fails on the rare value over 4000 bytes.
  [Details](#store-short-text-columns-as-varchar2-not-clob)
- **Keep the maintenance-window resource plan out of a load.** [Details](#keep-the-maintenance-window-resource-plan-out-of-a-load)
- **Pre-size undo, and let it grow in large steps.** [Details](#pre-size-undo-and-let-it-grow-in-large-steps)
- **Split memory between instances by measured benefit** when the datastore spans several instances on one host.
  [Details](#split-memory-between-instances-by-measured-benefit-provisional)
- **Reserve 1 GiB huge pages before host memory fragments.** [Details](#6-reserve-1-gib-huge-pages-before-host-memory-fragments)

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

## Store short text columns as `VARCHAR2`, not `CLOB`

**Mechanism.** The standard Oracle schema declares several short text columns as `CLOB`. The values are short, so
they are stored in the row, but in a multibyte database character set such as `AL32UTF8` Oracle stores CLOB text as
UTF-16. That takes two bytes for every ASCII character, plus a LOB header per value. `LIB_FEAT.FELEM_VALUES` is in
every `LIB_FEAT` row, and `LIB_FEAT` is the largest feature table, so the CLOB encoding makes the table substantially
larger. Fewer rows then fit in each block and in the buffer cache, and a lookup that misses the cache reads more
blocks. The client also handles each CLOB value through a LOB locator rather than as a plain string. Table
compression does not remove this cost. Changing the character set does not either: only the column type does.

**Do this:** before creating the schema, edit the schema creation script so these columns are `VARCHAR2(4000)`
instead of `CLOB`:

| Table | Column |
|---|---|
| `LIB_FEAT` | `FELEM_VALUES` |
| `SYS_HW_CHECK` | `FELEM_VALUES` |
| `RES_ENT_OKEY` | `MATCH_KEY` |
| `RES_RELATE` | `MATCH_KEY` |

Leave the other `CLOB` columns as they are, including `RES_ENT.FEATURES`. Their values can be long.

**Make them `VARCHAR2(32767)` with `MAX_STRING_SIZE = EXTENDED`, not `VARCHAR2(4000)`.** Under the default
`MAX_STRING_SIZE = STANDARD`, a `VARCHAR2` holds at most 4000 **bytes**. The engine sets no length limit of its own on
these columns. On a large, varied dataset, a small number of `LIB_FEAT.FELEM_VALUES` values came out just over 4000
bytes. Each one failed its insert with `ORA-12899: value too large for column`, and every loader that met one stopped.
A sample load does not reliably find these values, because they are rare. `MAX_STRING_SIZE = EXTENDED` raises the
`VARCHAR2` limit to 32767 bytes. Without it, a `VARCHAR2(32767)` column cannot be declared.

Set `MAX_STRING_SIZE = EXTENDED` in the database (for a multitenant datastore, in each PDB) **before** creating the
schema. This is Oracle's documented procedure. The change cannot be reversed:

```sql
ALTER SESSION SET CONTAINER = <pdb>;
ALTER PLUGGABLE DATABASE <pdb> CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE <pdb> OPEN UPGRADE;
ALTER SYSTEM SET MAX_STRING_SIZE = EXTENDED SCOPE = BOTH;
@?/rdbms/admin/utl32k.sql
ALTER PLUGGABLE DATABASE <pdb> CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE <pdb> OPEN;
@?/rdbms/admin/utlrp.sql
```

Then create the schema (with the `VARCHAR2(4000)` columns above) and widen the four columns before loading. This is the
configuration we measured:

```sql
ALTER TABLE LIB_FEAT     MODIFY (FELEM_VALUES VARCHAR2(32767));
ALTER TABLE SYS_HW_CHECK MODIFY (FELEM_VALUES VARCHAR2(32767));
ALTER TABLE RES_ENT_OKEY MODIFY (MATCH_KEY    VARCHAR2(32767));
ALTER TABLE RES_RELATE   MODIFY (MATCH_KEY    VARCHAR2(32767));
```

**Avoid:**

- keeping `LIB_FEAT.FELEM_VALUES` as `CLOB` on a datastore you expect to grow large;
- `VARCHAR2(4000)` under `MAX_STRING_SIZE = STANDARD`. It works until the first value over 4000 bytes, which can arrive
  hours into a load.

**How to check:**

```sql
-- EXTENDED, in every database or PDB the engine connects to
SELECT value FROM v$parameter WHERE name = 'max_string_size';
-- the four columns are VARCHAR2(32767)
SELECT table_name, column_name, data_type, data_length
  FROM user_tab_columns
 WHERE (table_name, column_name) IN (('LIB_FEAT','FELEM_VALUES'), ('SYS_HW_CHECK','FELEM_VALUES'),
                                     ('RES_ENT_OKEY','MATCH_KEY'), ('RES_RELATE','MATCH_KEY'));
-- longest values in bytes, on a representative sample load (full scans: run them on a sample, not a live load)
SELECT MAX(LENGTHB(FELEM_VALUES)) FROM LIB_FEAT;
SELECT MAX(LENGTHB(MATCH_KEY)) FROM RES_ENT_OKEY;
SELECT MAX(LENGTHB(MATCH_KEY)) FROM RES_RELATE;
```

## ⭐ New repositories with Chinese, Japanese or Korean data: turn transliterated keys off _(v4.5+)_

**Mechanism.** For a name or address written in Chinese, Japanese or Korean script, SENZING® SDK 4.5 can build
candidate keys from two romanized forms:

- the **transliterated** form, a phonetic reading of the characters;
- the **translated** form, an English rendering from the engine's gazetteer. For example, `三菱商事` becomes
  "Mitsubishi".

With both turned on, each such record carries both sets of keys. Composite keys such as `NAMEREGION_KEY` pair every
name token with every city form, so their number grows faster than either set alone. The extra keys mean more
candidates per record, more comparisons, and more rows written to and read from the key tables. In our testing,
records in these scripts cost several times the CPU per record of Latin-script records.

**Scope.** The setting has no effect on:

- Latin or Cyrillic names, which produced exactly the same keys and cost with it on or off;
- person names in any script, which keep their transliterated keys;
- a name or address part the gazetteer cannot translate, which keeps its transliterated keys so that it is never left
  without any.

Only translated organization names and translated road and city names lose their transliterated keys.

**Do this:** in a **new** repository whose data includes Chinese, Japanese or Korean names or addresses, set these in
the `SETTINGS` section of the Senzing configuration before loading the first record:

```json
"NAME_TRANSLITERATED_KEYS": false,
"ADDR_TRANSLITERATED_KEYS": false,
"NAME_TRANSLATED_KEYS": true,
"ADDR_TRANSLATED_KEYS": true
```

A setting that is absent counts as `true`. The engine refuses a configuration with both `NAME_*` settings `false`.
On the same records, turning transliterated keys off cut the CPU and the number of candidates per CJK record to about
a third.

**Avoid:**

- changing the setting on a repository that is already loaded. Records already loaded keep the keys they were given.
  We have not measured a mixed repository.
- turning the translated keys off instead. The translated form is the one that matches the same organization written
  in English.

**How to check:** the configuration in use carries the settings:

```sql
SELECT CONFIG_DATA_ID,
       JSON_VALUE(CONFIG_DATA, '$.G2_CONFIG.SETTINGS.NAME_TRANSLITERATED_KEYS') AS name_transliterated,
       JSON_VALUE(CONFIG_DATA, '$.G2_CONFIG.SETTINGS.ADDR_TRANSLITERATED_KEYS') AS addr_transliterated
  FROM SYS_CFG;
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

# Memory and storage settings

> [!NOTE]
> The queries in this section read Active Session History (ASH) and the cache advisories. ASH and AWR require the
> Oracle Diagnostics Pack licence.

## Pre-size undo, and let it grow in large steps

**Mechanism.** Every Senzing write transaction generates undo. Under a load with many concurrent writers the undo
tablespace grows quickly, and each time its data file must extend, the sessions that need new undo space queue
behind the extension on `enq: HW - contention`. Data files created with a small autoextend step (a few MB) extend
very often, so the queueing recurs throughout the load. A large step makes extensions rare but does not remove
them, and each larger extension takes longer: in our testing an undo tablespace on a 1 GB step that was still
growing toward its working size still briefly queued the writers. Pre-sizing is what removes the waits; the large
step is the fallback for growth beyond it.

**Do this:** size the undo tablespace for a full load before loading (measure how large it grows on a
representative load), and set a large autoextend step for anything beyond that. For a bigfile undo tablespace, in
the PDB when local undo is on:

```sql
ALTER DATABASE DATAFILE '<undo datafile>' RESIZE <size>G;
ALTER DATABASE DATAFILE '<undo datafile>' AUTOEXTEND ON NEXT 1G MAXSIZE UNLIMITED;
```

A smallfile data file cannot grow past about 32 GB with 8 KB blocks: for a smallfile undo tablespace, add data
files (`ALTER TABLESPACE <undo tablespace> ADD DATAFILE ... SIZE ...G AUTOEXTEND ON NEXT 1G`) instead.

**Avoid:** relying on autoextend alone during a load, and above all leaving the undo data file on a small autoextend
step.

**How to check:**

```sql
-- size and autoextend step of the undo data files
SELECT f.file_name, ROUND(f.bytes/1073741824, 1) AS gb,
       ROUND(f.increment_by * t.block_size / 1048576) AS next_mb, t.bigfile
  FROM dba_data_files f JOIN dba_tablespaces t ON t.tablespace_name = f.tablespace_name
 WHERE t.contents = 'UNDO';
-- high-water-mark waits by tablespace over the last day (p2 is the tablespace number). The AWR copy of ASH covers a
-- whole load but keeps only one sample in ten, so it can miss a burst of a few minutes entirely. Run the same query
-- on v$active_session_history (no sample_time filter) as well, while the instance's recent history still covers
-- the start of the load, when undo grows fastest.
SELECT ts.name, COUNT(*) AS samples
  FROM dba_hist_active_sess_history h
  LEFT JOIN v$tablespace ts ON ts.ts# = h.p2 AND ts.con_id = h.con_id
 WHERE h.event = 'enq: HW - contention' AND h.sample_time > SYSTIMESTAMP - INTERVAL '1' DAY
 GROUP BY ts.name ORDER BY 2 DESC;
```

The undo tablespace should not appear in the second query while it stays within its pre-sized space.

## Split memory between instances by measured benefit _(provisional)_

**Mechanism.** The Senzing datastore can be split across several databases (for example the core tables in one,
the feature tables in others), and when those run as several Oracle instances on one host they share one memory
budget. Our testing so far shows the instances do not benefit equally from extra buffer cache, so an even split, or
one by table size, can leave avoidable physical reads. _Provisional: we have measured the advice, not yet the effect
of a re-split._

**Do this:** after a representative load at scale, read `V$DB_CACHE_ADVICE` on each instance. Its estimates are
cumulative since the instance started, so compare only instances that have been up over the same load. For each
instance, take the reads (and read time) saved per extra GB just above the current size (`size_factor` 1), and move
memory towards the instance where that saving is largest: set `sga_target` and `sga_max_size` in the SPFILE
(`SCOPE=SPFILE`) and restart the container (see [section 5](#5-restart-with-the-container-not-with-shutdown--startup)).
Keep the total within the reserved huge pages (see
[section 6](#6-reserve-1-gib-huge-pages-before-host-memory-fragments)).

**Avoid:** splitting by table size alone.

**How to check:**

```sql
-- estimated physical reads and read time saved per extra GB at each candidate size (default 8 KB pool)
SELECT ROUND(size_for_estimate / 1024, 1) AS size_gb, size_factor, estd_physical_reads, estd_physical_read_time,
       ROUND((estd_physical_reads - LEAD(estd_physical_reads) OVER (ORDER BY size_for_estimate))
             / ((LEAD(size_for_estimate) OVER (ORDER BY size_for_estimate) - size_for_estimate) / 1024)) AS reads_saved_per_gb,
       ROUND((estd_physical_read_time - LEAD(estd_physical_read_time) OVER (ORDER BY size_for_estimate))
             / ((LEAD(size_for_estimate) OVER (ORDER BY size_for_estimate) - size_for_estimate) / 1024)) AS read_s_saved_per_gb
  FROM v$db_cache_advice
 WHERE name = 'DEFAULT' AND block_size = 8192 AND advice_status = 'ON'
 ORDER BY size_for_estimate;
```

Compare the `size_factor` 1 row across instances. After the next representative load, the instance that gained
memory should show fewer physical reads (`physical reads` in `V$SYSSTAT`, or the AWR Load Profile) for the same work.

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

## 6. Reserve 1 GiB huge pages before host memory fragments

A 1 GiB huge page needs 1 GiB of contiguous physical memory. On a host that has been running for a while, memory
is fragmented, and raising `vm.nr_hugepages` at that point fills only part of the request — without an error.
Dropping caches and compacting memory did not recover the rest for us: memory the kernel cannot move still splits
the free regions. The instances then start with part of the SGA on 4 KiB pages (`use_large_pages = TRUE`) unless the SGA is reduced to fit.

**Do this:** reserve the 1 GiB huge pages at boot on the kernel command line (for example
`default_hugepagesz=1G hugepagesz=1G hugepages=<n>`), or as early after boot as possible, before the page cache
and other workloads have used the memory. Reserve enough for every instance's SGA plus a small margin.

**Avoid:** expecting a runtime increase of `vm.nr_hugepages` on a busy host to reach the requested count, and
letting part of an SGA fall back to 4 KiB pages: with thousands of database sessions, each process then maps that
part of the SGA through its own page tables.

**How to check:**

```sh
# pages actually reserved per NUMA node — compare with what you asked for
grep -H . /sys/devices/system/node/node*/hugepages/hugepages-1048576kB/nr_hugepages
grep -E 'HugePages_(Total|Free)' /proc/meminfo
```

and, after each instance starts, the large-page summary in the startup messages of its alert log shows the pages it
expected and the pages it allocated: they should be equal.
