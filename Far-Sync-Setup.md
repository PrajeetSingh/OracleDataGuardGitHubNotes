# Far Sync Setup (Oracle 19c)

This runbook describes how to add a Far Sync instance into an existing Data Guard Broker configuration without rebuilding the environment.
Far Sync provides zero-data-loss protection by receiving SYNC redo from the primary and forwarding it to one or more standbys.

```plaintext
Primary (cdbapp1_sec)
        |
        | SYNC (AFFIRM)
        v
   Far Sync (FSInstScd)
        |
        | ASYNC (NOAFFIRM)
        v
 Standby (cdbapp1_mcd)
```

## Prerequisites
* Primary and standby databases already configured in Data Guard Broker.
* Flashback Database enabled on primary and standby.
* Password files identical across all nodes.
* Network connectivity verified between Primary ↔ Far Sync and Far Sync ↔ Standby.
* Oracle 19c binaries available.

This is a end-to-end runbook for a Broker-managed Far Sync addition.

## Phase 1: OS Preparation (As root)

Far Sync does not have a database (no datafiles, no SYSTEM tablespace), we only need the RDBMS binaries.

### 1. Install Pre-installation RPM
This is the fastest way to configure kernel parameters, limits, and create the oracle user on OEL7.

```Bash
yum install -y oracle-database-preinstall-19c
```

### 2. Create Directories

```Bash
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
mkdir -p /u01/app/oraInventory
chown -R oracle:oinstall /u01/app/
chmod -R 775 /u01/app/
```

### 3. Set Environment Variables

Add these to the /home/oracle/.bash_profile:

```Bash
#### ORACLE SETTINGS START #####
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.0.0/dbhome_1
export ORACLE_SID=FSInstScd
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
#### ORACLE SETTINGS END ####
```

## Phase 2: Software Installation (As oracle)

Oracle 19c uses Image-based installation.

### 1. Unzip the Binaries

```Bash
cd $ORACLE_HOME
unzip -q /path/to/LINUX.X64_193000_db_home.zip
```

### 2. Run the Installer

Run the installer in "Software Only" mode.

Option A: Interactive (GUI): If you have X11 forwarding:

```Bash
./runInstaller
```
* Configuration Option: Select "Set Up Software Only".

* Database Installation Options: Select "Single instance database installation".

* Database Edition: Select "Enterprise Edition" (Required for Far Sync).

Option B: Silent Mode (Command Line): If you don't have a GUI, use this response file approach:

```Bash
./runInstaller -silent -ignorePrereq -waitforcompletion \
oracle.install.option=INSTALL_DB_SWONLY \
UNIX_GROUP_NAME=oinstall \
INVENTORY_LOCATION=/u01/app/oraInventory \
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1 \
ORACLE_BASE=/u01/app/oracle \
oracle.install.db.InstallEdition=EE \
oracle.install.db.OSDBA_GROUP=dba \
oracle.install.db.OSOPER_GROUP=oper \
oracle.install.db.OSBACKUPDBA_GROUP=backupdba \
oracle.install.db.OSDGDBA_GROUP=dgdba \
oracle.install.db.OSKMDBA_GROUP=kmdba \
oracle.install.db.OSRACDBA_GROUP=racdba
```

### Phase 3: Post-Installation (As root)

Once the installer finishes, it will prompt you to run two scripts:

```Bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/oracle/product/19.0.0/dbhome_1/root.sh
```

### Phase 4: Final Verification for Far Sync

Check that the oracle binary is correctly linked and ready:

```Bash
cd $ORACLE_HOME/bin
./sqlplus -v
# Output should show: SQL*Plus: Release 19.0.0.0.0 - Production
```

Now that the software is installed, the next step is to configure the Listener and TNS so the Primary can communicate with this new node.


## Phase 5: Preparation (Far Sync VM)

### 1. Network & Password Synchronization

For a Far Sync instance, the network configuration is the most common point of failure. Because the Primary database will be shipping redo in SYNC mode, any network "hiccups" or timeouts will cause the Primary to hang.

**TNS Entry:** Ensure identical tnsnames.ora on primary, standby, and Far Sync.

Ensure this is identical on all nodes to facilitate role transitions.

Path: `$ORACLE_HOME/network/admin/tnsnames.ora`

```
cdbapp1_sec =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <Primary_IP>)(PORT = 1521))
    (CONNECT_DATA = (SERVICE_NAME = cdbapp1_sec_svc))
  )

cdbapp1_mcd =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <Standby_IP>)(PORT = 1521))
    (CONNECT_DATA = (SERVICE_NAME = cdbapp1_mcd_svc))
  )

FSInstScd =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <FarSync_IP>)(PORT = 1521))
    (CONNECT_DATA = (SERVICE_NAME = FSInstScd_SVC))
  )
```

**Static Listener:** Add the FSInstScd SID to listener.ora on the Far Sync VM, so the Broker can reach it even when the instance is down.

> [!NOTE]
> This is mandatory for the Broker to manage the instance.

Path: `$ORACLE_HOME/network/admin/listener.ora`

```plaintext

# Listener for Far Sync VM
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = <FarSync_IP>)(PORT = 1521))
    )
  )

# Static registration is MANDATORY for Broker management
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = FSInstScd_SVC)  # Service Name used in Connect Identifier
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = FSInstScd)           # Far Sync SID
    )
  )

```

**SQLNET Tuning:** (`sqlnet.ora`): Prevent network hangs.

Because we are using SYNC transport from the Primary to this Far Sync VM, network stability is critical. We add timeouts to ensure a hung network doesn't hang our Primary database. Values are in seconds.

Path: `$ORACLE_HOME/network/admin/sqlnet.ora`

```plaintext
# Security and Timeouts
SQLNET.OUTBOUND_CONNECT_TIMEOUT = 10
SQLNET.RECV_TIMEOUT = 10
SQLNET.SEND_TIMEOUT = 10

# TCP Keepalive to prevent firewall drops on long-standing DG connections
TCP.CONNECT_TIMEOUT = 10
```

**Password File:** Copy the password file from the Primary to the Far Sync VM's $ORACLE_HOME/dbs/ and rename it to orapwFSInstScd.

```bash
cp oracle@<primary_ip>:/u01/app/oracle/product/19.0.0/dbhome_1/dbs/orapwcdbapp1_sec $ORACLE_HOME/dbs/orapwFSInstScd
# Change paths and file names as per the environment.
```

### 2. Create Far Sync Control File (On Primary)

```sql
ALTER DATABASE CREATE FAR SYNC INSTANCE CONTROLFILE AS '/tmp/fs_control.ctl';
-- Transfer this to /u01/app/oracle/oradata/FSInstScd/ on the Far Sync VM.
```

### 3. Initialize Far Sync (On Far Sync VM)

#### a. Create a PFILE initFSInstScd.ora:

```Plaintext
DB_NAME=cdbapp1
DB_UNIQUE_NAME=FSInstScd
CONTROL_FILES='/u01/app/oracle/oradata/FSInstScd/fs_control.ctl'
DG_BROKER_START=TRUE
REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE
```

#### b. Mount the instance:

```bash
export ORACLE_SID=FSInstScd
sqlplus / as sysdba
```

```sql
STARTUP MOUNT PFILE='/path/to/initFSInstScd.ora';
CREATE SPFILE FROM PFILE;
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
SHOW PARAMETER SPFILE;
SELECT PROCESS, STATUS, CLIENT_PROCESS, SEQUENCE# FROM V$MANAGED_STANDBY;
```

## Phase 6: Adding to the Broker (On Primary)

Since your Broker is already running, connect to DGMGRL on the Primary:

### 1. Register the Far Sync Instance

```SQL
ADD FAR_SYNC 'FSInstScd' AS CONNECT IDENTIFIER IS 'FSInstScd';
```

### 2. Configure Redo Routes

Route redo from Primary to Far Sync (SYNC) with priority 1. Fall back to sending directly to the Standby (ASYNC) as priority 2 if Far Sync fails.

This defines the new "Primary --> Far Sync --> Standby" pipeline.

```SQL
-- Route redo from Primary to Far Sync (SYNC)
EDIT DATABASE 'cdbapp1_sec' SET PROPERTY RedoRoutes = '(LOCAL : (FSInstScd SYNC PRIORITY=1, cdbapp1_mcd ASYNC PRIORITY=2))';

-- Route redo from Far Sync to Standby (ASYNC)
EDIT FAR_SYNC 'FSInstScd' SET PROPERTY RedoRoutes = '(cdbapp1_sec : cdbapp1_mcd ASYNC)';
```

### 3. Enable the New Setup

```SQL
ENABLE FAR_SYNC 'FSInstScd';
```

### 4. Set NetTimeout 

We need to protect the Primary from stalling if the VM gets overwhelmed.

On the Primary, reduce the wait time so the database doesn't hang if the Far Sync VM lags. Here, we are setting it to wait not more than 10 seconds.

```SQL
EDIT DATABASE 'cdbapp1_sec' SET PROPERTY NetTimeout = 10;
```

## Phase 7: Final Sync & Validation

### 1. Add Standby Redo Logs (SRLs)

Add SRLs on the Far Sync VM to match the Primary's log size.

*Rule: Total SRLs = (Number of Primary Redo Log Groups + 1) * Number of Threads.*

```SQL
-- Repeat to match calculation above
ALTER DATABASE ADD STANDBY LOGFILE SIZE 200M; 

SELECT GROUP#, TYPE, MEMBER FROM V$LOGFILE WHERE TYPE = 'STANDBY';
```

### 2. Validate the Configuration

From DGMGRL:

```SQL
SHOW CONFIGURATION;
VALIDATE DATABASE 'cdbapp1_sec';
VALIDATE FAR_SYNC 'FSInstScd';
VALIDATE DATABASE 'cdbapp1_mcd';
```

### 3. Check the Transport

On the Primary, check that it is now shipping to the Far Sync:

```SQL
SELECT DEST_ID, DEST_NAME, STATUS, TARGET FROM V$ARCHIVE_DEST_STATUS WHERE STATUS <> 'INACTIVE';
```

## Phase 8: Monitoring & Troubleshooting

Monitoring Far Sync requires checking three layers:

1. Primary → Far Sync (SYNC transport)

2. Far Sync → Standby (ASYNC transport)

3. End-to-end apply on the standby

Use these scripts to ensure the Far Sync relay isn't introducing latency into Primary database.

### 1. Check Transport Lag (Primary -> Far Sync)

This confirms if the SYNC transport to the Far Sync VM is keeping up with the Primary's REDO generation.

Run below on Primary:

```sql
SET LINESIZE 200 PAGESIZE 2000
COLUMN destination FORMAT a15
COLUMN status FORMAT a10
COLUMN error FORMAT a20

SELECT dest_id, status, target, archived_seq#, applied_seq#, error 
FROM v$archive_dest_status 
WHERE dest_id IN (1, 2);

-- OR in Data Guard
SHOW DATABASE <primary_database_name>;
SHOW DATABASE cdbapp1_sec;
SHOW DATABASE <standby_database_name>;
SHOW DATABASE cdbapp1_mcd;
```

### 2. Verify REDO Relay (On Far Sync VM)

Since Far Sync has no data, it only shows the "Received" and "Sent" status.

```sql
-- Check RFS (Receiving from Primary) and LNS (Sending to Standby)
SELECT process, status, thread#, sequence#, block#, blocks 
FROM v$managed_standby 
WHERE process IN ('RFS', 'LNS', 'TT00');

-- OR in Data Guard
SHOW FAR_SYNC FSInstScd;
VALIDATE FAR_SYNC FSInstScd;
```

> [!NOTE]
> `TT00` is a helper process that manages redo transport timing, retries, and state transitions.

### 3. Check End-to-End Apply Lag (On Standby)

It tells you how far behind the Standby is from the Primary.

Run below on Standby:

```sql
SELECT name, value, datum_time, time_computed 
FROM v$dataguard_stats 
WHERE name IN ('transport lag', 'apply lag');

-- OR in Data Guard
SHOW DATABASE <standby_database_name>;
SHOW DATABASE cdbapp1_mcd;
```

### 4. Check Switchover Readiness

```sql
SELECT database_role, open_mode, protection_mode, protection_level FROM v$database;
SELECT switchover_status FROM v$database;
SELECT GROUP#, TYPE, MEMBER FROM V$LOGFILE WHERE TYPE = 'STANDBY';

-- Or in Data Guard Broker, check if `Ready for Switchover` is `Yes`.
VALIDATE DATABASE <standby_database_name>;
VALIDATE DATABASE cdbapp1_mcd;
SHOW CONFIGURATION;
```

### 5. Check Overall Data Guard Health

```sql
SELECT message, timestamp 
FROM v$dataguard_status 
ORDER BY timestamp DESC;

-- OR in Data Guard
SHOW CONFIGURATION;
VALIDATE DATABASE cdbapp1_sec;
VALIDATE FAR_SYNC FSInstScd;
VALIDATE DATABASE cdbapp1_mcd;
```

### 6. Common Far Sync Warning Codes

| Warning   | Meaning                           | Typical Cause                         | 
| ---       | ---                               | ---                                 | 
| ORA-16857 | Member disconnected from redo source | Network hiccup, NetTimeout too low  | 
| ORA-16855 | Transport lag exceeded threshold | Slow network or FS VM CPU pressure  | 
| ORA-16853 | Apply lag exceeded threshold      | Standby I/O or MRP lag               | 
| ORA-16086 | Far Sync cannot write to SRL      | Missing SRLs, wrong VALID_FOR, wrong DB_UNIQUE_NAME | 
| ORA-16766 | Redo transport error              | Incorrect RedoRoutes or TNS issues   | 


## Key Considerations

**Zero Data Loss:** Since you are using SYNC to the Far Sync, any CPU or Network bottleneck on the Far Sync VM will impact Primary performance.

**Instance Recovery:** Even though there are no datafiles, Far Sync uses the CONTROL_FILES and SRLs. Ensure these are on fast storage.

**Failover Scenario:** If the Far Sync VM goes down, the Broker will attempt to route redo directly to the Standby (likely in ASYNC mode) depending on the Priority 2 rule set in your RedoRoutes configuration.

**RAC Environment:** When adding SRLs to the Far Sync instance, remember that the Far Sync instance must have enough SRLs to handle the redo from the Primary. A good rule of thumb is:

`Total SRLs = (Number of Online Redo Log Groups on Primary + 1) * Number of Threads.`

If you have a 2-node RAC Primary, you'll need to account for both threads on the single Far Sync instance.

https://database-heartbeat.com/2023/01/30/far-sync-step-by-step/

