# Setup 2 Node Oracle RAC as Standby for Standalone Primary using backup of Standalone Standby DB

This task will be done in three phases:

* **Phase 1:** The 2-Node RAC Build
* **Phase 2:** RMAN backup of Standalone Standby database
* **Phase 3:** RAC Standby Creation & Far Sync Integration

`Phase1` will be configuration of Virtualbox VMs and Oracle RAC installation. For Oracle RAC configuration we can use below link.

https://github.com/PrajeetSingh/OracleRACGitHubKB/blob/main/Installing%20and%20Configuring%20Oracle%20RAC.md

`Phase 2` will be taking the backup from our existing Physical Standby (mcd1). It completely offloads the I/O and CPU hit from the Primary database (scd1). We will use RMAN Backup Set Level 0 (full backup) of the database and archived logs, and generate Standby Control File from mcd1.

In `Phase 3`, once the backup is ready and the RAC softwae is installed, we will:

* Restore the RMAN backup of `mcd1` (Standalone Standby) into the ASM shared storage on the new RAC.

* Register the new database in the clusterware (`srvctl`).

* Add the new RAC database to our existing Data Guard Broker Configuration (Standalone Primary, Standby and Far Sync we created earlier).

* Update the `RedoRoutes` so the Far Sync instance (`FSInstdScd`) knows to split the redo stream and ship to both `mcd` and the new RAC standby simultaneously.


## Phase 1

The blueprint for getting the VirtualBox built.

### 1. VM Sizing & Provisioning

* Two identical Virtual Machines (e.g., racnode1 and racnode2).

* RAM: Minimum 8 GB per node (12 GB+ is ideal, so the Grid Infrastructure doesn't crawl).

* CPU: At least 4 vCPUs per node.

* OS Disk: 70 GB dynamically allocated per node for the Linux OS and Oracle software binaries.

### 2. The Network Stack (The Most Critical Part)

We need to configure three network adapters for each VM before you install the OS:

* **Adapter 1 (NAT):** Used strictly for internet access so we can run yum update and download prerequisite packages.

* **Adapter 2 (Host-Only Adapter):** This is our Public Network. It will handle our node's Public IP, the Virtual IP (VIP), and the SCAN cluster addresses.

* **Adapter 3 (Internal Network):** This is our Private Interconnect. Oracle uses this for the cluster heartbeat and Cache Fusion. Name the internal network something like rac-interconnect in VirtualBox.

### 3. Shared Storage for ASM

RAC requires shared storage, so that both nodes can read and write simultaneously. Do below to create shared disks in Virtualbox:

1. Open VirtualBox **Virtual Media Manager**. Go to `Media` on left hand side menu.

2. Create three new `.vdi` disks. **They MUST be "Fixed size", not dynamically allocated.**

    * `asm_grid.vdi` (approx. 5 GB) - For the OCR and Voting disks. 
    
    I would recommend it around 200 GB for office because in office we should use GIMR, which is not required in home lab.

    * `asm_data.vdi` (approx. 50 GB) - For the database files.

    * `asm_fra.vdi` (approx. 40 GB) - For the Fast Recovery Area and archive logs.

3. Once created, select each disk in the Media Manager, click **Properties**, and change the type to **Shareable**. If you skip this, Node 2 will lock up or corrupt Node 1's writes.

4. Attach all three shared disks to `racnode1` and `racnode2`.

Once the Virtual hardware is ready, the next step is planning the network layout and installing Oracle Linux (OEL7 here).


Here is a standard, clean IP allocation matrix you can use for your VirtualBox network adapters.

### 4. Oracle RAC IP Address Layout

#### Network Subnets
* **Public Network (Host-Only Adapter):** `192.168.56.x`
* **Private Network (Internal Network):** `10.10.10.x`

---

#### Existing Data Guard Environment

```
192.168.56.110  scd1.localdomain scd1
192.168.56.111  scd2.localdomain scd2
192.168.56.112  mcd1.localdomain mcd1
192.168.56.113  mcd2.localdomain mcd2
```

---

#### Node 1 (`racnode1`)
* **Public IP:** `192.168.56.101`
* **Virtual IP (VIP):** `192.168.56.111`
* **Private IP (Interconnect):** `10.10.10.101`

#### Node 2 (`racnode2`)
* **Public IP:** `192.168.56.102`
* **Virtual IP (VIP):** `192.168.56.112`
* **Private IP (Interconnect):** `10.10.10.102`

---

#### SCAN (Single Client Access Name)
* **SCAN Name:** `rac-scan`
* **SCAN IPs:** `192.168.56.121`, `192.168.56.122`, `192.168.56.123`

> **Note:** For a home lab without a dedicated DNS server, you can cheat and just map a single SCAN IP in your `/etc/hosts` file to get past the installer warnings.

#### `/etc/hosts` file for the lab

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

# --- Existing Data Guard Environment ---
192.168.56.110  scd1.localdomain scd1
192.168.56.111  scd2.localdomain scd2
192.168.56.112  mcd1.localdomain mcd1
192.168.56.113  mcd2.localdomain mcd2

# --- New Oracle RAC Public Network ---
192.168.56.121  racnode1.localdomain racnode1
192.168.56.122  racnode2.localdomain racnode2

# --- New Oracle RAC Virtual IPs (VIP) ---
192.168.56.131  racnode1-vip.localdomain racnode1-vip
192.168.56.132  racnode2-vip.localdomain racnode2-vip

# --- New Oracle RAC SCAN ---
# (Using a single IP to bypass DNS requirements in the lab)
192.168.56.141  rac-scan.localdomain rac-scan

# --- New Oracle RAC Private Interconnect ---
# (Routable only between racnode1 and racnode2)
10.10.10.121    racnode1-priv.localdomain racnode1-priv
10.10.10.122    racnode2-priv.localdomain racnode2-priv
```
Once you have the VMs created, the disks attached, and the OS installed with these IPs, we will need to configure udev rules so Oracle can actually see those disks as raw ASM devices.

