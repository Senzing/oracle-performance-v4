# Test environment

The hardware, operating systems and software behind the findings on the [Oracle performance](README.md) page. It
is listed as context, not as a benchmark: which levers matter most depends on ratios (working set against
buffer cache, statements per record), so a system with a different shape can rank them differently.

All hosts are on one LAN. Datastore and broker traffic run on a dedicated 10 GbE data-plane network; management
traffic is on a separate interface and subnet.

## Database server (one host, Oracle only — no Senzing processes)

| | |
|---|---|
| Chassis | Supermicro `SYS-221H-TN24R` (2U, 24 × U.2/U.3 NVMe bays) |
| CPU | 2 × Intel Xeon Gold 6438Y+ (Sapphire Rapids), 64 cores / 128 threads, 2 NUMA nodes |
| Memory | 1 TiB DDR5-4800 ECC; 360 × 1 GiB huge pages reserved at boot |
| Data volume | 23 × Micron 7450 PRO 1.92 TB U.3 NVMe, Linux `md` RAID 0, 128 KiB chunk; `ext4`, `noatime,data=writeback`. Holds the data files of all three instances. |
| Redo volume | 1 × Micron 7450 PRO 1.92 TB NVMe, `ext4`, `noatime,data=writeback`. Holds the online redo logs of all three instances. |
| Network | 10 GbE |
| OS | Ubuntu 26.04.1 LTS, kernel 7.0 |
| Containers | Docker 29.1.3 |
| Network interrupts | `irqbalance` off; the interface's 119 queue interrupts bound one per CPU across **both** NUMA nodes (set at runtime — see the main page). Before this, the driver had bound all of them to the card's own node. |
| NUMA | automatic NUMA balancing on (kernel default); huge pages split evenly across the two nodes |

## Application servers (Senzing SDK processes only)

| Hosts | CPU | Memory | OS |
|---|---|---|---|
| 2 × Dell PowerEdge R650xs | 2 × Intel Xeon Gold 6326 (Ice Lake), 32 cores / 64 threads, 2 NUMA nodes | 512 GB DDR4-3200 ECC | Ubuntu 24.04.5 LTS, kernel 6.8 |
| 1 × Dell PowerEdge R640 | 2 × Intel Xeon Platinum 8280M (Cascade Lake), 56 cores / 112 threads, 2 NUMA nodes | 512 GB ECC | Ubuntu 26.04.1 LTS, kernel 7.0 |

Each has a 10 GbE data-plane interface and runs Docker 29.1.3. In total: 120 cores / 240 threads.

## Software

| Component | Version / setting |
|---|---|
| Database | Oracle AI Database 26ai Enterprise Edition 23.26.1.0, the Oracle container image `container-registry.oracle.com/database/enterprise:23.26.1.0` |
| Senzing | SENZING® SDK 4.5.0 |
| Loader | Senzing combined consumer: each process reads records from the broker, calls `add_record`, and also processes redo (`REDO_PERCENT` 20). 67 processes × 12 threads = 804 engine threads (18 + 18 + 31 processes across the three application servers). |
| Message broker | Apache ActiveMQ Artemis 2.57.0, on the database server |
| Test data | 100 million synthetic records, one data source, published to the broker as JSON |

### Senzing engine settings

```json
"ER": { "ENABLE_DEFERRED_WRITE": "1", "ENTITY_LOCK_MODE": "LEASE",
        "ENABLE_RES_ENT_FEATURE_CACHE": "1", "WORK_RETRY_TIMEOUT": "90" }
```

`RES_ENT.FEATURES` is added to the schema before the first engine initialization (see the main page).

## Datastore layout

The Senzing datastore is split across **three Oracle instances** on the database server, one container and one
pluggable database each, using the Senzing engine's multi-node (`HYBRID`) configuration:

| Instance | Senzing tables | `SGA_TARGET` | `PGA_AGGREGATE_TARGET` |
|---|---|---|---|
| core | all tables except the three below | 80 GB | 32 GB |
| res | `RES_FEAT_EKEY`, `RES_FEAT_STAT` (+ `SYS_VARS`) | 135 GB | 32 GB |
| libfeat | `LIB_FEAT` (+ `SYS_VARS`) | 125 GB | 32 GB |

Settings common to all three instances:

| Setting | Value |
|---|---|
| Block size | 8 KB |
| Senzing tablespace | bigfile, automatic segment space management, system-managed extents |
| Online redo | 4 groups × 16 GB, one member each, on the redo volume |
| `filesystemio_options` | `SETALL` (direct and asynchronous IO) |
| `use_large_pages` | `TRUE` (SGA on 1 GiB huge pages) |
| `processes` / `open_cursors` | 10240 / 2000 |
| `cursor_sharing` | `EXACT` |
| `optimizer_adaptive_statistics` | `FALSE` (default) |
| `resource_manager_plan` | `FORCE:` in the CDB and the PDB (see the main page) |
| Optimizer statistics | `METHOD_OPT` = `FOR ALL COLUMNS SIZE 1`; representative statistics, locked (see the main page) |

Each instance's container is started with `--shm-size=32g`, `--cap-add IPC_LOCK`, `--cap-add SYS_NICE`,
`--group-add` for the huge-page group, `--ulimit memlock=-1` and `--ulimit nofile=1048576`, with the data
and redo directories bind-mounted from the two volumes above. No CPU or memory limits are set on any container.

## Method

Every run starts from an empty datastore (purged through the SDK), loads the same records, and
changes one setting at a time. Oracle's Automatic Workload Repository (AWR) reports, `sar` on every host and the
Oracle dynamic performance views are captured throughout.
