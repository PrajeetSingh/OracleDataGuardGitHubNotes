# Introduction to Oracle Standby Databases

* High availability technique
* It supports homogeneous replication
* It is recommended to have Primary and Standby on same version and patch level

## Basic Requirements for Home Lab

* 2 Virtual Box VMs with same OS (Linux)
* Primary DB should be ready
* On Second Node, create Standby DB using Level 0 backup of the Primary DB
* Then configure Data Guard and DG Broker between Primary and Standby
* If we want to configure Far Sync server too, then it is not mandatory (in home lab) but recommended to have a separate VM for it

## Types of Standby Databases

* Physical Standby Database: Redo Apply
* Logical Stanbdy Database: SQL Apply
* Snapshot Standby Database: To facilitate testing on Standby Database in Read Write mode

## Modes of Standby Databases

* **Maximum Performance:** It is Async / no affirm mode. Primary will ship redo logs to Standby but will not wait for acknowledgement. So if there is any network issue, it'll not impact performance of Primary Database.

* **Maximum Protection:** It is Sync / affirm mode. Primary will ship redo logs to Standby and will wait for acknowledgement from Standby. It will ensures that data reached at Standby location but if there is a network issue, it can hang or put Primary in wait state until it receives acknowledgement from Standby.

* **Maximum Availability:** It is async / sync / affirm mode. It is a mix of *Maximum Performance* and *Maximum Protection*. When network is fine, it works in *Maximum Protection* mode but if network breaks, it switches to *Maximum Performance* mode.

Above modes are set on `log_archive_dest_2` on Primary database.

**High Level Process:** `Primary DB` --writes changes to--> `LGWR` --Transmit Redo through Redo Stream to--> `Standby Redo Logs` --MRP Proceess applies Redos to--> `Standby DB`.

Till Oracle 11gR1, LGWR created archived log files, those arcived log files were shipped to Standby DB server, then applied on Standby DB but Oracle 11gR2 onwards, instead of archived log files, these are passed to Standby DB through Redo Stream. It reduced the Time Delay. Also it resolves the problem that Now archived logs are used for *Gap Recovery* only.

Redo Log Streaming method is called, `Real Time Apply`.

## Architecture

1. `LNS process` of Primary database captures Redo from Redo Log buffer.
2. Sends it to `RFS process` of Standby database through `Oracle net` (Listeners and TNS entries on Primary and Standby Sites).
3. `RFS process` then writes that redo information to `Standby Redo log files`.
4. If `LNS process` is not fast enough to caputre redo information before it goes to `Online Redo Log files` or if redo data are going to online redo log files very quickly then LNS process will read from Online Redo Log files and send redo to `RFS process` through `Oracle Net`.
5. If some network outage occur and online redo log gets log switch and data goes to archived redo log files, before its been written to Standby Redo Log files, then `RFS process` will directly communicate to `ARCn process` and works for `Archive Log Gap resolution`.

Processes involved above are: `LGWR`, `LNSn`, `RFS`, `MRP` (`LSP` for Logical Standby) and `ARCn`.

### Processes on Primary Database Server

**LGWR (Log Writer):** It collects transaction redo information and updates online redo files
**LNS (Log writer Network Server):** LGWR passes redo to LNS process, which transfers data directly to RFS process on the standby database.
**ARCn (Archiver):** Creates a copy of the Online Redo Log files. It is also responsible for shipping redo data to an RFS process at a Standby Database.

### Processes on Standby Database Server

**RFS (Remote File Server):** It can get Redo Data either from LNS process or from ARCn process of Primary, then writes REDO information to Standby Redo Log files. Each LNS and ARCn process that communicates with Standby Database, has its own RFS process.
**ARCn (Archiver):** It archives Standby Redo Logs
**MRP (Managed Recovery Process):** It applies archived log information to the Physical Standby DB.
**LSP (Logical Stabdby):** It applies archived log information to the Logical Standby DB.

> [!IMPORTANT]
> FAL_SERVER and FAL_CLIENT servers are used for Gap Recovery. FAL_CLIENT is deprecated now. FAL stands for Fetch Archive Logs.
> Standby REDO Logs are N+1 of Primary REDO Logs i.e., if Primary has 3 REDO Logs, then there should be 4 Standby REDO Logs.

## Parameters Configured in Data Guard

**log_archive_config:** Enables or disables the sending of redo logs to remote destinations and specifies the `db_unique_name` fro all databases in Data Guard configuration.

```sql
alter system set log_archive_config='DG_CONFIG=(pimrary_db,standby_db)' scope=both;
```

**log_archive_dest_:** Specifies the local and remote destinations for archived redo logs.

```sql
alter system set log_archive_dest_1='LOCATION=/u05/app/oracle/arch VALID_FOR(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=primary_db' scope=both;

alter system set log_archive_dest_2='SERVICE=standby_db LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=standby_db' scope=both;
```

**log_archive_dest_2:** Contains TNS Alias or TNS entry of Standby Database, also contains async, no affirm, timeout, db unique name, etc. This parameter should be set to null `''` before we configure Data Guard. It is then set by Data Guard Broker.
**log_archive_dest_state_n:** It should be enabled or disabled to start or stop shipping of redo logs to Standby or local locations.

```sql
alter system set log_archie_dest_state_1='ENABLE' scope=both;
```

**db_file_name_convert:** Specifies the conversion parameters for filenames between the Primary and Standby databases.

```sql
alter system set db_file_name_convert='/primary/db/location','/standby/db/location' scope=both;
```

**log_file_name_convert:** (<directory structure in Primary>,<directory structure in Standby>)

```sql
alter system set log_file_name_convert='/primary/db/location','/standby/db/location' scope=both;
```

**fal_server:**: TNS Alias/Entry of Primary DB.

```sql
alter system set FAL_SERVER='primary_db' scope=both;
-- Set it on both Primary and Standby for corresponding servers
```

**fal_client:**: TNS Alias/Entry of Standby DB
**db_name:** It must be same on Primary and Standby DBs.

**db_unique_name:** It must be different on Primary and Standby DBs.

```sql
alter system set db_unique_name='primary_db' scope=both;
alter system set db_unique_name='standby_db' scope=both;
```

**standby_file_management:** It must be set to `auto`. It ensures that data file is added on Standby too when it is added on Primary.

```sql
alter system set standby_file_management='auto' scope=both;
```

> [!IMPORTANT]
> Ideally, Standby DB file locations should be exact copy of Primary DB but they are different, we need to use `db_file_name_convert` and `log_file_name_convert`.
> Also, Primary Database must be in Force Logging Mode, to ensure logging is never in NoLogging mode for any object. It is required by Standby DB.

```sql
show paramter spfile
show parameter log_archive_config
show parameter log_archive_dest_1
show parameter log_archive_dest_state_1
select log_mode, force_logging from v$database;
alter database force logging;
```

> [!NOTE]
> oracledbwr.com and oracle-base.com are good references to read further or take any help.
> https://oracle-base.com/articles/19c/data-guard-setup-using-broker-19c