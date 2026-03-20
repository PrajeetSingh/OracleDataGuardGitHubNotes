# 🛠️ RAC Physical Standby Build Runbook

![Oracle](https://img.shields.io/badge/Oracle-19c-red)
![Data Guard](https://img.shields.io/badge/Data%20Guard-Physical%20Standby-blue)
![RAC](https://img.shields.io/badge/RAC-2--Node-green)
![Status](https://img.shields.io/badge/Runbook-Production%20Ready-success)
![Maintained](https://img.shields.io/badge/Maintained-Yes-brightgreen)

---

## 📌 Overview

This runbook describes how to:

> Clone a standalone standby database into a **2-node Oracle RAC Physical Standby** using **RMAN backup/restore**, **ASM**, and **Data Guard Broker**, routed via **Far Sync**.

---

## 🧭 Table of Contents

* [🏗️ Architecture](#️-architecture)
* [💽 ASM Layout](#-asm-layout)
* [⚙️ Phase 1 – Primary Prep & Donor Validation](#️-phase-1--primary-prep--donor-validation)
* [💾 Phase 2 – RMAN Backup (Donor)](#-phase-2--rman-backup-donor)
* [🌐 Phase 3 – Network & Listener Setup](#-phase-3--network--listener-setup)
* [⚙️ Phase 4 – RAC Parameter & ASM Setup](#️-phase-4--rac-parameter--asm-setup)
* [📦 Phase 5 – RMAN Restore to ASM](#-phase-5--rman-restore-to-asm)
* [🔁 Phase 6 – Redo Log Configuration](#-phase-6--redo-log-configuration)
* [🧩 Phase 7 – Clusterware Registration](#-phase-7--clusterware-registration)
* [🔗 Phase 8 – Data Guard Broker Integration](#-phase-8--data-guard-broker-integration)
* [✅ Phase 9 – Final Validation](#-phase-9--final-validation)
* [⚠️ Critical Checks](#️-critical-checks)

---

## 🏗️ Architecture

| Role      | Host           | DB Name      | Notes       |
| --------- | -------------- | ------------ | ----------- |
| Primary   | scd1standalone | cdbapp1_sec  | Source      |
| Far Sync  | scd2farsync    | FSInstScd    | Redo router |
| Standby 1 | mcd1standalone | cdbapp1_mcd  | Donor       |
| Standby 2 | racnode1/2     | cdbapp1_mrac | Target RAC  |

---

## 💽 ASM Layout

| Disk Group | Purpose                        |
| ---------- | ------------------------------ |
| `+DATA`    | Datafiles, SPFILE, Password    |
| `+FRA`     | Recovery & Redo Logs           |
| `+GRID`    | OCR/Voting (Do NOT use for DB) |

---

# ⚙️ Phase 1 – Primary Prep & Donor Validation

## On Primary

```sql
CREATE UNDO TABLESPACE undotbs2 DATAFILE '/u01/app/oracle/oradata/CDBAPP1/undotbs02.dbf' SIZE 1G AUTOEXTEND ON NEXT 100M MAXSIZE 10G;
```

```sql
SELECT tablespace_name, contents 
FROM dba_tablespaces 
WHERE tablespace_name IN ('UNDOTBS1','UNDOTBS2');
```

```sql
ALTER SYSTEM SWITCH LOGFILE;
-- Run multiple times
```

---

## On Donor Standby

```sql
SELECT tablespace_name FROM dba_tablespaces WHERE contents='UNDO';
```

---

# 💾 Phase 2 – RMAN Backup (Donor)

## Stop Apply

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
EDIT DATABASE cdbapp1_mcd SET STATE='APPLY-OFF';
SELECT process, status FROM v$managed_standby;
```

## Backup

```bash
mkdir -p /tmp/stby_bkp
rman 
connect target /
```

```rman
RUN {
  ALLOCATE CHANNEL c1 DEVICE TYPE DISK;
  ALLOCATE CHANNEL c2 DEVICE TYPE DISK;
  BACKUP AS COMPRESSED BACKUPSET ARCHIVELOG ALL NOT BACKED UP 1 TIMES FORMAT '/u01/stby_bkp/al_pre_%U.bkp';
  BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL 0 DATABASE FORMAT '/u01/stby_bkp/db_L0_%U.bkp';
  BACKUP CURRENT CONTROLFILE FOR STANDBY FORMAT '/u01/stby_bkp/ctl_%U.bkp';
  BACKUP SPFILE FORMAT '/u01/stby_bkp/spfile_%U.bkp';
  BACKUP AS COMPRESSED BACKUPSET ARCHIVELOG ALL FORMAT '/u01/stby_bkp/al_post_%U.bkp';
  RELEASE CHANNEL c1;
  RELEASE CHANNEL c2;
}

LIST BACKUP SUMMARY;
LIST BACKUP OF DATABASE;
LIST BACKUP OF ARCHIVELOG ALL;
LIST BACKUP OF CONTROLFILE;
LIST BACKUP OF SPFILE;

VALIDATE BACKUPSET;
VALIDATE DATABASE;
LIST BACKUP OF ARCHIVELOG ALL;

CROSSCHECK BACKUP;
RESTORE DATABASE PREVIEW;
EXIT;
```

```SQL
-- In SQL, note the highest received/applied sequence before and after.
SELECT thread#, MAX(sequence#) FROM v$archived_log GROUP BY thread# ORDER BY thread#;
```

```rman
BACKUP AS COMPRESSED BACKUPSET ARCHIVELOG ALL FORMAT '/u01/stby_bkp/al_final_%U.bkp';

RESTORE DATABASE PREVIEW;
```

## Resume Apply

```sql
CREATE PFILE='/tmp/stby_bkp/initcdbapp1_mrac.ora' FROM SPFILE;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
EDIT DATABASE cdbapp1_mcd SET STATE='APPLY-ON';
EXIT;
```

## Ship to RAC Node 1

```bash
ssh oracle@racnode1 "mkdir -p /u01/stby_bkp"

rsync -avhP /u01/stby_bkp/ oracle@racnode1:/u01/stby_bkp/
rsync -avhP $ORACLE_HOME/dbs/orapwcdbapp1 oracle@racnode1:/u01/orapwcdbapp1_mrac
```

---

# 🌐 Phase 3 – Network & Listener Setup

## listener.ora (both nodes)

```ini
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = cdbapp1_mrac_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = cdbapp11)
    )
  )
```

```bash
lsnrctl reload
```

---

## tnsnames.ora (all nodes)

```ini
cdbapp1_mrac =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = rac-scan.localdomain)(PORT = 1521))
    (CONNECT_DATA = (SERVICE_NAME = cdbapp1_mrac_DGMGRL))
  )
```

---

# ⚙️ Phase 4 – RAC Parameter & ASM Setup

```ini
*.db_name='cdbapp1'
*.db_unique_name='cdbapp1_mrac'
*.cluster_database=TRUE
*.db_create_file_dest='+DATA'
*.db_recovery_file_dest='+FRA'
*.db_recovery_file_dest_size='30G'
*.standby_file_management='AUTO'
*.remote_login_passwordfile='EXCLUSIVE'
```

---

# 📦 Phase 5 – RMAN Restore to ASM

```rman
RESTORE STANDBY CONTROLFILE FROM '/tmp/stby_bkp/ctl_XXXX.bkp';
ALTER DATABASE MOUNT;

RUN {
  SET NEWNAME FOR DATABASE TO '+DATA';
  RESTORE DATABASE;
  SWITCH DATAFILE ALL;
  RECOVER DATABASE;
}
```

---

# 🔁 Phase 6 – Redo Log Configuration

```sql
ALTER DATABASE ADD LOGFILE THREAD 1 GROUP 11 ('+DATA','+FRA') SIZE 200M;
ALTER DATABASE ENABLE PUBLIC THREAD 2;
```

---

# 🧩 Phase 7 – Clusterware Registration

```bash
srvctl add database -db cdbapp1_mrac -oraclehome $ORACLE_HOME \
-dbtype RAC -role PHYSICAL_STANDBY -startoption MOUNT \
-spfile +DATA/CDBAPP1_MRAC/spfilecdbapp1_mrac.ora
```

---

# 🔗 Phase 8 – Data Guard Broker Integration

```bash
dgmgrl /
```

```sql
ADD DATABASE cdbapp1_mrac AS CONNECT IDENTIFIER IS cdbapp1_mrac MAINTAINED AS PHYSICAL;
ENABLE DATABASE cdbapp1_mrac;
EDIT DATABASE cdbapp1_mrac SET STATE='APPLY-ON';
```

---

# ✅ Phase 9 – Final Validation

```bash
srvctl start instance -db cdbapp1_mrac -instance cdbapp12
```

```sql
SELECT inst_id, instance_name, status FROM gv$instance;
```

---

# ⚠️ Critical Checks

### 🔴 MUST VERIFY

* UNDO tablespaces exist for both RAC threads
* Database role = `PHYSICAL STANDBY`
* TNS resolution works from **ALL nodes**
* Broker shows **SUCCESS**

---

### 🚫 NEVER DO

* ❌ Drop CURRENT redo logs
* ❌ Use `+GRID` for database files
* ❌ Skip standby redo logs

---

### ✅ BEST PRACTICES

* ✔ Always validate after each phase
* ✔ Keep backup copy until build is complete
* ✔ Test switchover readiness

---
