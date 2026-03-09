# Oracle 19c RAC Installation and Configuration Guide

This guide details the end-to-end installation and configuration of a 2-node Oracle Real Application Clusters (RAC) 19c environment on Oracle Linux 7.9. 

This runbook assumes the base Virtual Machines (`racnode1`) and shared storage (`asm_grid`, `asm_data`, `asm_fra`) have already been provisioned.

---

## Phase 1: Operating System Prerequisites

Before configuring ASM or installing Grid Infrastructure, the OS must be prepared with the required Oracle packages, users, and kernel parameters. Run these steps on `racnode1` as the `root` user.

### 1. Install the Oracle Preinstall Package

This package automatically downloads required dependencies, creates the `oracle` user, sets up the `oinstall` and `dba` groups, and configures kernel parameters (`sysctl.conf`).

```bash
yum update -y
yum install -y oracle-database-preinstall-19c
```

### 2. Create Grid Infrastructure Groups and User

The preinstall package does not create the specific groups or user required for ASM and Grid Infrastructure. We must create them manually.

```bash
# Create ASM specific groups
groupadd -g 54327 asmdba
groupadd -g 54328 asmoper
groupadd -g 54329 asmadmin

# Add the oracle user to the asmdba group so it can access ASM storage
usermod -a -G asmdba oracle

# Create the grid user with oinstall as primary, and ASM groups as secondary
useradd -u 54331 -g oinstall -G asmadmin,asmdba,asmoper,dba grid
```

### 3. Set Passwords for Oracle and Grid Users

```bash
passwd oracle
passwd grid
```

### 4. Create the Installation Directories

```bash
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/grid
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1

# Assign Grid directories to the grid user
chown -R grid:oinstall /u01/app/19.0.0/grid
chown -R grid:oinstall /u01/app/grid

# Assign Oracle directories to the oracle user
chown -R oracle:oinstall /u01/app/oracle

# Set base directory permissions
chmod -R 775 /u01
```

## Phase 2: Configuring Shared Storage (UDEV Rules)

Oracle Clusterware requires raw block devices for ASM. We use udev rules to ensure the oracle user owns the disks and that the mapping survives reboots.

### 1. Identify SCSI IDs
Retrieve the unique hardware SCSI IDs for your three shared disks (sdb, sdc, sdd).

```bash
/usr/lib/udev/scsi_id -g -u -d /dev/sdb
/usr/lib/udev/scsi_id -g -u -d /dev/sdc
/usr/lib/udev/scsi_id -g -u -d /dev/sdd
```

### 2. Create the UDEV Rules File

```Bash
vi /etc/udev/rules.d/99-oracle-asmdevices.rules
```

Insert the following lines, replacing <INSERT_SCSI_ID> with the actual outputs from Step 1.

```plaintext
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VB47c55dae-ea554058", SYMLINK+="oracleasm/asm_grid", OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VB3090c68d-2bc699b3", SYMLINK+="oracleasm/asm_data", OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VB1e6c019e-c86743b4", SYMLINK+="oracleasm/asm_fra", OWNER="grid", GROUP="asmadmin", MODE="0660"
```

### 3. Partition Disks and Reload Rules

Create a single primary partition on each disk, then reload udev to apply the permissions.

```bash
echo -e "n\np\n1\n\n\nw" | fdisk /dev/sdb
echo -e "n\np\n1\n\n\nw" | fdisk /dev/sdc
echo -e "n\np\n1\n\n\nw" | fdisk /dev/sdd

partprobe

-- Ignore below error
-- Warning: Unable to open /dev/sr0 read-write (Read-only file system).  /dev/sr0 has been opened read-only.

udevadm control --reload-rules
udevadm trigger


```

Verify the disks appear with correct ownership (oracle:dba):

```bash
ls -al /dev/oracleasm/
```

## Phase 3: Cloning to Node 2 (`racnode2`)

To ensure identical configurations, clone `racnode1` to create `racnode2`.

1. Power off `racnode1`.

2. In VirtualBox, right-click `racnode1` -> Clone.

3. Choose **Full Clone** and select **Generate new MAC addresses for all network adapters**. Name `racnode2`.

4. Storage check

* Open the Settings for racnode2 in VirtualBox and go to Storage.

* Look at the SATA controller. If VirtualBox created copies (e.g., asm_grid_clone.vdi), remove them.

* Click the "Add Hard Disk" icon and attach the exact same original asm_grid.vdi, asm_data.vdi, and asm_fra.vdi files that racnode1 is using.

* Once those original shared disks are attached, we are good to power on racnode2.

5. Boot `racnode2` and update the hostname and IP addresses:

  * Change Hostname to `racnode2.localdomain` (`hostnamectl set-hostname racnode2.localdomain`).

  * Update the Public IP (`enp0s8`) to `.122`.

  ```bash
  vi /etc/sysconfig/network-scripts/ifcfg-enp0s8
  IPADDR=192.168.56.122
  ```

  * Update the Private IP (`enp0s9`) to `.122`.

  ```bash
  vi /etc/sysconfig/network-scripts/ifcfg-enp0s9
  IPADDR=10.10.10.
  
  systemctl restart network
  ```

  * Update `/etc/hosts` File

```bash
vi /etc/hosts

# Public IPs
192.168.56.121    racnode1.localdomain      racnode1
192.168.56.122    racnode2.localdomain      racnode2

# Private IPs (Interconnect)
10.10.10.121      racnode1-priv.localdomain racnode1-priv
10.10.10.122      racnode2-priv.localdomain racnode2-priv

# Virtual IPs (VIPs)
192.168.56.131    racnode1-vip.localdomain  racnode1-vip
192.168.56.132    racnode2-vip.localdomain  racnode2-vip

# SCAN IPs (Round-robin resolution)
192.168.56.140    rac-scan.localdomain      rac-scan
192.168.56.141    rac-scan.localdomain      rac-scan
192.168.56.142    rac-scan.localdomain      rac-scan
```

6. Check connectivity

```bash
ping -c 3 racnode1
ping -c 3 racnode1-priv
```

7. Reboot `racnode2` and verify `ping` connectivity between both nodes on all interfaces.

## Phase 4: Setup SSH Conntectivity

We need to set up passwordless SSH for both the grid and oracle users across both nodes. They need to be able to SSH into the other node, and also into themselves.

### 1. Configure SSH for the grid user

Run below commands on both `racnode1` and `racnode2` as grid user:

```bash
su - grid
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

ssh-copy-id racnode1
ssh-copy-id racnode2

ssh racnode1 date
ssh racnode2 date
```

### 2. Configure SSH for the oracle user

Run below commands on both `racnode1` and `racnode2` as oracle user:

```bash
su - oracle
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

ssh-copy-id racnode1
ssh-copy-id racnode2

ssh racnode1 date
ssh racnode2 date
```

With the network pinging and SSH equivalence completely done for both users, the OS layer is officially complete and we have a pristine, clustered foundation.

## Phase 5: Unzip the Grid Infrastructure binaries

In 19c, we unzip the Grid Infrastructure software directly into the Grid Home directory, rather than running it from a temporary staging folder (unlike 12c).

### 1. Unzip the Grid Infrastructure Binary

Log into racnode1 VirtualBox desktop GUI as Grid user and open a terminal (or su to grid user).

```bash
su - grid
cd /u01/app/19.0.0/grid

unzip -q /tmp/LINUX.X64_193000_grid_home.zip
```

### 2. Install the cvuqdisk RPM

Before we launch the installer, Oracle Clusterware needs the cvuqdisk package installed on both nodes so the Cluster Verification Utility (CVU) can discover and validate the shared ASM disks.

As the root user on racnode1, install it from the newly unzipped Grid Home directory.

```bash
su - root
export CVUQDISK_GRP=asmadmin

-- RPM version here may differ
rpm -ivh /u01/app/19.0.0/grid/cv/rpm/cvuqdisk-1.0.10-1.rpm

scp /u01/app/19.0.0/grid/cv/rpm/cvuqdisk-1.0.10-1.rpm racnode2:/tmp/
ssh racnode2 "export CVUQDISK_GRP=asmadmin; rpm -ivh /tmp/cvuqdisk-1.0.10-1.rpm"
```

## Phase 6: Grid Infrastructure Installation.

### Grid Infrastructure Wizard Walkthrough

### 1. Launch the Grid Setup Wizard

In office environment, to run the GUI, we may need `X11 forwarding` or `DISPLAY` variables.

Run below as Grid user on `racnode`:

```bash
cd /u01/app/19.0.0/grid
./gridSetup.sh
```

### 2. Configuration Option

* Select Configure Oracle Grid Infrastructure for a New Cluster.

* Click Next.

### 3. Cluster Configuration

* Select Configure an Oracle Standalone Cluster.

* Click Next.

### 4. Grid Plug and Play Information

* **Cluster Name:** rac-cluster (or a preferred name).

* **SCAN Name:** rac-scan.localdomain (This must perfectly match /etc/hosts entry).

* **SCAN Port:** 1521

* Uncheck `Configure GNS` (Grid Naming Service is not needed for this lab).

* Click Next.

### 5. Cluster Node Information

* We will see racnode1 already listed.

* Click Add...

  * Public Hostname: racnode2.localdomain

  * Virtual Hostname: racnode2-vip.localdomain

* Click OK.

* Click the SSH Connectivity... button. Enter the grid user password and click Setup. Even though we did this manually, letting the wizard verify it ensures the installer recognizes the keys.

* Click Next.

### 6. Network Interface Usage

The installer will detect our three network adapters. We must classify them correctly:

* enp0s3 (The 10.0.2.x NAT IP): Change the Use to Do Not Use. (If Oracle tries to route cluster traffic over the NAT adapter, the installation will fail).

* enp0s8 (The 192.168.56.x Host-Only IP): Change the Use to Public.

* enp0s9 (The 10.10.10.x Internal IP): Change the Use to ASM & Private.

  * In office environment, I would prefer ASM and Private on different NICs to keep ASM and Private Interconnect / Cache Fusion on separate traffic routes. This helps in troubleshooting issues and when Oracle Support is also involved in troubleshooting RAC issues.

* Click Next.

### 7. Storage Option Information

* Select Use Oracle Flex ASM for storage.

* Click Next.

### 8. Grid Infrastructure Management Repository (GIMR)

* Select No. (The GIMR consumes significant CPU and disk space; bypassing it is best practice for a lightweight VirtualBox lab. Though in Office environment, it is important).

* Click Next.

### 9. Create ASM Disk Group

* **Disk Group Name:** GRID

* **Redundancy:** Select External. (Because we only created one 5GB virtual disk for the Grid, high or normal redundancy will fail).

* **Allocation Unit Size:** 4 MB

* In the "Select Disks" window, check the box next to /dev/oracleasm/asm_grid.

> [!NOTE]
> If you don't see your disks, click "Change Discovery Path" and enter /dev/oracleasm/*.

* Click Next.

### 10. ASM Password

* Select `Use same passwords for these accounts`.

* Enter `super secret` password. Ignore the warning if it complains about complexity.

* Click Next.

### 11. Failure Isolation

* Select Do not use Intelligent Platform Management Interface (IPMI).

* Click Next.

### 12. Root Script Execution

While we can automate this, it is standard practice to run these manually to monitor the output for errors.

* Leave `Automatically run configuration scripts` unchecked.

* Click Next.

### 13. Prerequisite Checks

The Cluster Verification Utility (CVU) will run. In a VirtualBox lab, it is entirely normal to get a few warnings regarding physical memory, swap size, or NTP/DNS setup.

If it flags anything, check the `Ignore All` box in the top right corner and `click Next`.

### 14. Summary & Install

* Review the configuration summary.

* Click Install.

When installation pauses at around 70-80%, a prompt will appear asking to run two specific scripts as the root user (orainstRoot.sh and root.sh).

> [!WARNING]
> If we run root.sh concurrently on both nodes to save time, it will instantly corrupt the Oracle Cluster Registry (OCR) and Voting Disk creation, forcing a complete rebuild.

### 15. Execute Configuration Scripts (`root.sh` and `orainstRoot.sh`)

Run these two scripts as the `root` user.

> [!CRITICAL]
> These scripts must be executed strictly in order. We **must** wait for `root.sh` to finish completely on `racnode1` before executing it on `racnode2`. Running them concurrently will cause a fatal cluster corruption. I know I am repeating it after `WARNING` but it is Critical.

**Step A: Execute on Node 1 (`racnode1`)**

Open a new terminal session, switch to the `root` user, and run:

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

> [!NOTE]
> The root.sh script will take several minutes. Watch the output carefully. Do not proceed to Node 2 until you see the message: `Configure Oracle Grid Infrastructure for a Cluster ... succeeded`.

**Step B: Execute on Node 2 (`racnode2`)**

Once Node 1 is 100% finished, open a terminal on racnode2, switch to the root user, and run the exact same scripts:

```Bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

**Step C: Finalize the Installation**

After Node 2 outputs its success message, return to the Grid Setup Wizard GUI on `racnode1` and click `OK`.

Grid Infrastructure is now live. We can verify the cluster status as the grid user by running:

```bash
crsctl stat res -t
```

If `crsctl stat res -t` shows all cluster resources (like the VIPs, SCAN listener, and ASM instances) online, we have a fully functional Oracle Grid.


## Phase 7: Oracle Database 19c Software-Only installation (as `oracle` user)

Now that the clusterware is managing the nodes and the ASM storage is online, we just need to lay down the Oracle Database binaries.

Again, we need to unzip the database software directly into the Oracle Home directory, rather than a staging folder.

> [!TIP]
> Usually in a RAC environment, it is highly recommended to install the Database software first ("Software Only" mode) and then use the Database Configuration Assistant (DBCA) to create the actual database later. This separates the binary layout from the database creation process.

### 1. Unzip the Database Binary

Log into the `racnode1` desktop GUI as `oracle` user and open a terminal.

```bash
su - oracle
cd /u01/app/oracle/product/19.0.0/dbhome_1
unzip -q /tmp/LINUX.X64_193000_db_home.zip
```

### 2. Launch the Oracle Universal Installer (OUI)

```bash
./runInstaller
```

### 3. Database Setup Wizard Walkthrough

Follow these exact steps in the graphical installer.

* **Configuration Option**

  * Select Set Up Software Only.
  * Click Next.

* **Database Installation Options**

  * Select Oracle Real Application Clusters database installation.
  * Click Next.

* **Node Selection**

  * Both `racnode1` and `racnode2` should appear and be selected.
  * Click the `SSH Connectivity` button. Enter the oracle user password and click Setup to let the installer verify the SSH equivalence we built earlier.
  * Click Next.

* **Database Edition**

  * Select Enterprise Edition.
  * Click Next.

* **Installation Location**

  * **Oracle Base:** Should automatically populate as `/u01/app/oracle`.
  * **Software Location:** Should automatically populate as `/u01/app/oracle/product/19.0.0/dbhome_1`.
  * Click Next.

* **Operating System Groups**

  * The `oracle-database-preinstall-19c` package created these for us. Verify the mappings:
    * Database Administrator (OSDBA): dba
    * Database Operator (OSOPER): oper
    * Database Backup and Recovery (OSBACKUPDBA): backupdba
    * Data Guard (OSDGDBA): dgdba
    * Transparent Data Encryption (OSKMDBA): kmdba
    * Real Application Clusters (OSRACDBA): racdba
* Click Next.

* **Root Script Execution**

  * Leave `Automatically run configuration scripts` unchecked. (We will run this manually later, just like the Grid installation).
  * Click Next.

* **Prerequisite Checks**

  * Ignore any home lab specific warnings (like swap space or physical memory) by checking `Ignore All`.
  * Click Next.

* **Summary & Install**
  
  * Review the configuration.
  * Click Install.

### 4. Execute the root.sh Script

When the progress bar reaches the end, the installer will prompt to run root.sh.

> [!NOTE]
> Unlike the Grid installation, this script is much faster and less destructive, but we must still run it sequentially.

**Execute on Node 1 (`racnode1`) and Node 2 (`racnode2`) sequentially**

Open a terminal, switch to the root user, and run below first on Node 1 (`racnode1`), then Node 2 (`racnode2`):

```bash
/u01/app/oracle/product/19.0.0/dbhome_1/root.sh
```

Return to the OUI wizard on `racnode1`, click `OK` on the script prompt, and then click `Close` once the success screen appears.

The Oracle 19c Database binaries are now successfully deployed across both nodes!

The final step will be creating ASM Disk Groups `+DATA` and `+FRA`. 

## Phase 6: Creating ASM Disk Groups (`ASMCA` or `SQL` Method)

During the Grid Infrastructure installation, only the `+GRID` disk group was created to hold the Oracle Cluster Registry (OCR) and Voting Disks. We must now use ASMCA to create the `+DATA` (for datafiles) and `+FRA` (for the Fast Recovery Area and Archivelogs) disk groups across the cluster.

### 1. Connect to the ASM Instance as `grid` user

On **`racnode1`**, log in as the `grid` user, set your environment to the local ASM instance (`+ASM1`), and connect via SQL*Plus.

```bash
su - grid
export ORACLE_SID=+ASM1
export ORACLE_HOME=/u01/app/19.0.0/grid
export PATH=$ORACLE_HOME/bin:$PATH

sqlplus / as sysasm
```

### 2. Create the Disk Groups via SQL

Execute the following commands to create the disk groups using the raw block devices we secured with udev rules. We will use `EXTERNAL REDUNDANCY`.

```sql
-- Create the DATA disk group for database files
CREATE DISKGROUP DATA EXTERNAL REDUNDANCY 
DISK '/dev/oracleasm/asm_data' 
ATTRIBUTE 'compatible.asm' = '19.0', 'compatible.rdbms' = '19.0';

-- Create the FRA disk group for recovery and archivelogs
CREATE DISKGROUP FRA EXTERNAL REDUNDANCY 
DISK '/dev/oracleasm/asm_fra' 
ATTRIBUTE 'compatible.asm' = '19.0', 'compatible.rdbms' = '19.0';

exit;
```

### 3. Mount Disk Groups Across the Cluster

Because we created these manually via SQL on Node 1, we must tell Oracle Clusterware to mount them on Node 2 (`racnode2`). Use the `srvctl` cluster utility to start the disk groups globally.

```bash
srvctl start diskgroup -diskgroup DATA
srvctl start diskgroup -diskgroup FRA
```

### 4. Verify Cluster Storage

```bash
crsctl stat res -t | grep dgs

# We should see ora.DATA.dg, ora.FRA.dg, and ora.GRID.dg all listed as ONLINE on both racnode1 and racnode2.
```