# Far Sync Setup 

**Assumption:** We have a functioning Data Guard Broker configuration, we need to inject the Far Sync instance into the existing environment rather than rebuilding it.

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
mkdir -p /u01/app/oracle/product/19.3.0/dbhome_1
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

Oracle 19c uses Image-based installation. You must unzip the software directly into the ORACLE_HOME.

### 1. Unzip the Binaries

Upload the LINUX.X64_193000_db_home.zip to the VM, then:

```Bash
cd $ORACLE_HOME
unzip -q /path/to/LINUX.X64_193000_db_home.zip
```

### 2. Run the Installer

We will run the installer in "Software Only" mode.

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
ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1 \
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
/u01/app/oracle/product/19.3.0/dbhome_1/root.sh
```

### Phase 4: Final Verification for Far Sync

Check that the oracle binary is correctly linked and ready:

```Bash
cd $ORACLE_HOME/bin
./sqlplus -v
# Output should show: SQL*Plus: Release 19.0.0.0.0 - Production
```

Now that the software is installed, the next logical step is to configure the Listener and TNS so the Primary can communicate with this new node.


## Phase 5: Preparation (Far Sync VM)

### 1. Network & Password Synchronization

For a Far Sync instance, the network configuration is the most common point of failure. Because the Primary database will be shipping redo in SYNC mode, any network "hiccups" or timeouts will cause the Primary to hang.

**TNS Entry:** Ensure PROD_DB, STDBY_DB, and FSInstScd are in the tnsnames.ora on all three nodes.

Ensure this is identical on all nodes to facilitate role transitions.

Path: `$ORACLE_HOME/network/admin/tnsnames.ora`

```
PROD_DB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <Primary_IP>)(PORT = 1521))
    (CONNECT_DATA = (SERVICE_NAME = PROD_DB_SVC))
  )

STDBY_DB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <Standby_IP>)(PORT = 1521))
    (CONNECT_DATA = (SERVICE_NAME = STDBY_DB_SVC))
  )

FSInstScd =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <FarSync_IP>)(PORT = 1521))
    (CONNECT_DATA = (SERVICE_NAME = FSInstScd_SVC))
  )
```

**Static Listener:** Add the FSInstScd SID to listener.ora on the Far Sync VM. This is mandatory for the Broker to manage the instance.

On the Far Sync VM, add this to listener.ora so the Broker can reach it even when the instance is down:

Path: `$ORACLE_HOME/network/admin/listener.ora`

```plaintext

# Listener for Far Sync VM
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.56.111)(PORT = 1521))
    )
  )

# Static registration is MANDATORY for Broker management
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = FSInstScd_SVC)  # Service Name used in Connect Identifier
      (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
      (SID_NAME = FSInstScd)           # Your Far Sync SID
    )
  )

```

**SQLNET Tuning:** (`sqlnet.ora`)

Because we are using SYNC transport from the Primary to this VM, network stability is critical. We add timeouts to ensure a hung network doesn't hang our Primary database.

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

cp orapwPROD_DB orapwFSInstScd

### 2. Create Far Sync Control File (On Primary)

```sql
ALTER DATABASE CREATE FAR SYNC INSTANCE CONTROLFILE AS '/tmp/fs_control.ctl';
-- Transfer this to /u01/app/oracle/oradata/FSInstScd/ on the Far Sync VM.
```

### 3. Initialize Far Sync (On Far Sync VM)

Create a PFILE initFSInstScd.ora:

```Plaintext
DB_NAME=PROD_DB
DB_UNIQUE_NAME=FSInstScd
CONTROL_FILES='/u01/app/oracle/oradata/FSInstScd/fs_control.ctl'
DG_BROKER_START=TRUE
REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE
```

Mount the instance:

```bash
export ORACLE_SID=FSInstScd
sqlplus / as sysdba
STARTUP MOUNT PFILE='/path/to/initFSInstScd.ora';
CREATE SPFILE FROM PFILE;
shutdown immediate;
startup mount;
show parameter spfile;
SELECT PROCESS, STATUS, CLIENT_PROCESS, SEQUENCE# FROM V$MANAGED_STANDBY;
```

## Phase 6: Adding to the Broker (On Primary)

Since your Broker is already running, connect to DGMGRL on the Primary:

### 1. Register the Far Sync Instance

```SQL
ADD FAR_SYNC 'FSInstScd' AS CONNECT IDENTIFIER IS 'FSInstScd';
```

### 2. Configure Redo Routes

This is the most critical step. You must tell the Broker to route redo through the Far Sync to reach the Standby. Replace PROD_DB and STDBY_DB with your actual DB_UNIQUE_NAME values.

This defines the new "Primary --> Far Sync --> Standby" pipeline.

```SQL
-- Route redo from Primary to Far Sync (SYNC)
EDIT DATABASE 'cdbapp1_sec' SET PROPERTY RedoRoutes = '(LOCAL : FSInstScd SYNC)';

-- Route redo from Far Sync to Standby (ASYNC)
EDIT FAR_SYNC 'FSInstScd' SET PROPERTY RedoRoutes = '(cdbapp1_sec : cdbapp1_mcd ASYNC)';
```

### 3. Enable the New Setup

```SQL
ENABLE FAR_SYNC 'FSInstScd';
```

We need to protect the Primary from stalling if the VM gets overwhelmed.

### 4. Set NetTimeout 

On the Primary, reduce the wait time so the database doesn't hang if the Far Sync VM lags.

```SQL
EDIT DATABASE 'cdbapp1_sec' SET PROPERTY NetTimeout = 10;
```

### 5. Set FarSyncAdvise

This tells the Broker how to handle a Far Sync failure.

**Value:** `PREFER`: If Far Sync fails, the Primary will try to ship directly to the Standby (likely in ASYNC) to keep the Primary running.

```SQL
EDIT FAR_SYNC 'FSInstScd' SET PROPERTY FarSyncAdvise = 'PREFER';
```

## Phase 7: Final Sync & Validation

### 1. Add Standby Redo Logs (SRLs)

The Broker might warn you if SRLs are missing. Add them on the Far Sync VM to match the Primary's log size:

```SQL
ALTER DATABASE ADD STANDBY LOGFILE SIZE 200M; -- Repeat to match Primary + 1

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

Use these scripts to ensure the Far Sync relay isn't introducing latency into your Primary database.

### 1. Check Transport Lag (On Primary)

This confirms if the SYNC transport to the Far Sync VM is keeping up with the Primary's redo generation.

```sql
SET LINESIZE 200
COLUMN destination FORMAT a15
COLUMN status FORMAT a10
COLUMN error FORMAT a20

SELECT dest_id, status, target, archived_seq#, applied_seq#, error 
FROM v$archive_dest_status 
WHERE dest_id IN (1, 2);
```

### 2. Verify Redo Relay (On Far Sync VM)

Since Far Sync has no data, it only shows the "Received" and "Sent" status.

```sql
-- Check RFS (Receiving from Primary) and LNS (Sending to Standby)
SELECT process, status, thread#, sequence#, block#, blocks 
FROM v$managed_standby 
WHERE process IN ('RFS', 'LNS', 'TT00');
```

### 3. Check End-to-End Apply Lag (On Standby)

It tells you how far behind the Standby is from the Primary, accounting for the Far Sync hop.

```sql
SELECT name, value, datum_time, time_computed 
FROM v$dataguard_stats 
WHERE name IN ('transport lag', 'apply lag');
```

### 4. Check DG Health (if it is ready for switchover)

```sql
SELECT database_role, open_mode, protection_mode, protection_level FROM v$database;
SELECT switchover_status FROM v$database;
SELECT GROUP#, TYPE, MEMBER FROM V$LOGFILE WHERE TYPE = 'STANDBY';
```

## Key Considerations

**Zero Data Loss:** Since you are using SYNC to the Far Sync, any CPU or Network bottleneck on the Far Sync VM will impact Primary performance.

**Instance Recovery:** Even though there are no datafiles, Far Sync uses the CONTROL_FILES and SRLs. Ensure these are on fast storage.

**Failover Scenario:** If the Far Sync VM goes down, the Broker will attempt to route redo directly to the Standby (likely in ASYNC mode) depending on your FarSyncAdvise setting.

**RAC Environment:** When adding SRLs to the Far Sync instance, remember that the Far Sync instance must have enough SRLs to handle the redo from the Primary. A good rule of thumb is:

`Total SRLs = (Number of Online Redo Log Groups on Primary + 1) * Number of Threads.`

If you have a 2-node RAC Primary, you'll need to account for both threads on the single Far Sync instance.

https://database-heartbeat.com/2023/01/30/far-sync-step-by-step/

