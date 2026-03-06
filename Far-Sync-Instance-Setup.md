# Far Sync Instance Setup

This is the consolidated, Oracle 19c Data Guard Far Sync Master Runbook. It incorporates the manual setup, the TNS/Network configuration, and the Data Guard Broker (DGMGRL) implementation for professional-grade management.

## Phase 1: Environment & Network Preparation

Before the Far Sync instance can receive redo, the Primary needs to know it exists and how to talk to it.

**Add Static Listeners:** Ensure your listener.ora on the Far Sync VM has a static entry for the Far Sync SID (e.g., FARSync).

**TNS Configurations:** Add the Far Sync TNS alias to tnsnames.ora on all nodes (Primary, Standby, and Far Sync).

**Password File:** Copy the Oracle password file from the Primary to the Far Sync VM. Rename it to match the Far Sync SID.

Path: $ORACLE_HOME/dbs/orapw<FarSync_SID>

Perform these steps on all nodes: Primary, Standby, and your Far Sync VM.

### 1. Update tnsnames.ora

Ensure all three nodes can resolve each other.

```Plaintext
PROD_DB = (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=Primary_IP)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=PROD_DB_SVC)))
STDBY_DB = (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=Standby_IP)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=STDBY_DB_SVC)))
FS_DB = (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=FarSync_IP)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=FS_DB_SVC)))
```

### 2. Security & Listeners

**Static Listener:** On the Far Sync VM, add a static SID entry for FS_DB in listener.ora so the Broker can restart it remotely.

**Password File:** Copy orapw<Primary_SID> from Primary to Far Sync $ORACLE_HOME/dbs/ and rename it to orapwFS_DB.

## Phase 2: Create the Far Sync Instance

You cannot use a standard control file; you must create a specific "far sync" version from the Primary.

### 1. Generate Control File (On Primary)

```sql
ALTER DATABASE CREATE FAR SYNC CONTROLFILE AS '/tmp/farsync01.ctl';
-- Transfer this file to the Far Sync VM.
```

### 2. Create the PFILE (On Far Sync VM)

Far Sync instances don't have datafiles, but they do need an init.ora (PFILE) and Redo Logs.

Create a minimal initFS_DB.ora:

```plaintext

DB_NAME=PROD_DB
DB_UNIQUE_NAME=FS_DB
CONTROL_FILES='/u01/app/oracle/oradata/FS_DB/farsync01.ctl'
DG_CONFIG=(PROD_DB, STDBY_DB, FS_DB)
REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE

```

### 3. Mount the Instance

```Bash

sqlplus 
/ as sysdba
STARTUP MOUNT PFILE='/path/to/initFS_DB.ora';
# Create SPFILE from PFILE if desired
CREATE SPFILE FROM PFILE;

```

### 4. Add Standby Redo Logs (SRLs)

The Far Sync instance needs SRLs to receive the redo. They should be the same size as the Primary's online redo logs.

```SQL
ALTER DATABASE ADD STANDBY LOGFILE SIZE 200M; -- Repeat for n+1 count
```

## Phase 3: Data Guard Broker Configuration

This is the modern way to manage the redo routes.

### 1. Enable Broker (On All Nodes)

```SQL
ALTER SYSTEM SET dg_broker_start=TRUE;
```

### 2. Create Configuration (On Primary via DGMGRL)

```Bash
dgmgrl sys/password
```

```SQL
CREATE CONFIGURATION 'FS_Config' AS PRIMARY DATABASE IS 'PROD_DB' CONNECT IDENTIFIER IS 'PROD_DB';
ADD FAR_SYNC 'FS_DB' AS CONNECT IDENTIFIER IS 'FS_DB';
ADD DATABASE 'STDBY_DB' AS CONNECT IDENTIFIER IS 'STDBY_DB' MAINTAINED AS PHYSICAL;
```

### 3. Define the Redo Routes

This tells Oracle to send redo to the Far Sync, which then forwards it to the Standby.

```SQL
EDIT DATABASE 'PROD_DB' SET PROPERTY RedoRoutes = '(LOCAL : FS_DB SYNC)';
EDIT FAR_SYNC 'FS_DB' SET PROPERTY RedoRoutes = '(PROD_DB : STDBY_DB ASYNC)';
ENABLE CONFIGURATION;
```

## Phase 4: Health Check & Verification

### 1. Broker Validation

```SQL
SHOW CONFIGURATION;
VALIDATE DATABASE 'PROD_DB';
VALIDATE FAR_SYNC 'FS_DB';
```

### 2. Manual Redo Flow Check

Run this on the Far Sync VM to ensure it is actively relaying:

```SQL
SELECT PROCESS, STATUS, THREAD#, SEQUENCE# FROM V$MANAGED_STANDBY WHERE PROCESS IN ('RFS', 'LNS');

-- On the Far Sync Instance, check if redo is being received:
SELECT ARCHIVED, STATUS FROM V$MANAGED_STANDBY WHERE PROCESS LIKE 'RFS%';

-- On the Primary, check the destination status:
SELECT DEST_ID, STATUS, ERROR FROM V$ARCHIVE_DEST_STATUS WHERE DEST_ID=2;
```


## Operational Best Practices

**NetTimeout:** Set NetTimeout to 10–15 seconds on the Primary to prevent the database from hanging if the Far Sync VM becomes unresponsive during high load.

**No Datafiles:** Do not attempt to "restore" the database. Far Sync only manages the control file and redo logs.

**Licensing:** Far Sync requires the Active Data Guard license.

**Network:** Ensure the network latency between Primary and Far Sync is minimal (<10ms) to truly benefit from SYNC transport.

**Zero Data Loss:** In this `SYNC AFFIRM` setup, the Primary only commits once the Far Sync acknowledges receipt of the redo.
