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

⭐ marks a lever that made a large difference in our own v4 testing.

## At a glance — the key levers

_Added as levers are measured._

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
