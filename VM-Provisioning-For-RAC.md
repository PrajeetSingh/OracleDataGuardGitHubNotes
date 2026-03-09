# Provisioning an Oracle Linux VM in VirtualBox for Oracle RAC

This guide outlines the end-to-end process for creating and configuring an Oracle Linux 7.9 Virtual Machine (VM) in Oracle VirtualBox, specifically optimized to serve as a node in an Oracle 19c Real Application Clusters (RAC) environment.

## Prerequisites
* Oracle VirtualBox installed.
* Oracle Linux 7.9 ISO downloaded.
* Sufficient host resources (Minimum 8GB+ RAM and 2+ vCPUs recommended per RAC node).

---

## Phase 1: VM Creation and Hardware Allocation

1. Open VirtualBox and click **New**.
2. **Name and Operating System:**
   * Name: `racnode1` (or your preferred hostname)
   * Type: `Linux`
   * Version: `Oracle (64-bit)`
3. **Memory Size:** Allocate at least `8192 MB` (8 GB), ideally `12288 MB` (12 GB) if host resources permit.
4. **Processors:** Allocate at least `2` vCPUs.
5. **Hard Disk:** Create a virtual hard disk now.
   * File type: `VDI (VirtualBox Disk Image)`
   * Storage on physical hard disk: `Dynamically allocated`
   * Size: `70 GB` (This is strictly for the OS and Oracle binary installations).
   * Click **Create**.



---

## Phase 2: Configuring Shared Storage for Oracle ASM

Oracle RAC requires shared storage that all nodes can access simultaneously. We must create specific "Fixed Size" disks and mark them as "Shareable".

1. Go to **File -> Tools -> Virtual Media Manager**.
2. Click **Create** to make three new VDI disks. **Crucial: Select "Fixed size" for all of them.**
   * `asm_grid.vdi` (5 GB) - For OCR and Voting Disks.
   * `asm_data.vdi` (50 GB) - For Database Files.
   * `asm_fra.vdi` (40 GB) - For the Fast Recovery Area.
3. Once created, select each of the three `asm_` disks in the list, click **Properties**, and check the **Shareable** box. Apply the changes.
4. Go back to the main VirtualBox window, select your `racnode1` VM, and click **Settings -> Storage**.
5. Under **Controller: SATA**, click the "Add Hard Disk" icon and attach all three `asm_` disks you just created.
6. Under **Controller: IDE**, click the empty optical drive, click the disk icon on the right, and choose your **Oracle Linux 7.9 ISO** file.



---

## Phase 3: Network Adapter Configuration

Oracle RAC requires a highly specific network topology. We need three distinct network adapters.

1. In the VM **Settings**, go to **Network**.
2. **Adapter 1 (Internet Access):**
   * Enable Network Adapter: Checked
   * Attached to: `NAT`
   * *Purpose: Allows the VM to reach the internet for `yum updates` and package downloads.*
3. **Adapter 2 (Public Network & VIP):**
   * Enable Network Adapter: Checked
   * Attached to: `Host-only Adapter`
   * Name: Select your VirtualBox Host-Only Ethernet Adapter.
   * Advanced -> Promiscuous Mode: `Deny`
   * *Purpose: Handles the Public IP, Virtual IP (VIP), and SCAN traffic.*
4. **Adapter 3 (Private Interconnect):**
   * Enable Network Adapter: Checked
   * Attached to: `Internal Network`
   * Name: `rac-interconnect` (Type this in manually).
   * Advanced -> Promiscuous Mode: `Deny`
   * *Purpose: Dedicated backbone for Oracle Clusterware heartbeat and Cache Fusion traffic.*

---

## Phase 4: Oracle Linux OS Installation

1. Start the VM. Choose **Install Oracle Linux 7.9** from the boot menu.
2. Select your language and proceed to the **Installation Summary** screen.
3. **Software Selection:** Choose **Server with GUI**.
4. **Installation Destination:**
   * Select **ONLY** the 70 GB OS disk (`sda`).
   * Ensure the three ASM shared disks (`sdb`, `sdc`, `sdd`) are **unchecked**. The OS must not touch these.
   * Select **Automatically configure partitioning** and click **Done**.
5. **Network & Host Name:**
   * **Host Name:** Type `racnode1.localdomain` at the bottom and **click Apply**. (Do not skip clicking Apply!).
   * **enp0s3 (NAT):** Toggle to **ON**. (Leave as DHCP).
   * **enp0s8 (Host-Only):** Toggle to **ON**. Click **Configure...** -> **IPv4 Settings** -> Method: **Manual**. Add your designated Public IP (e.g., `192.168.56.121`) and Netmask (`255.255.255.0`).
   * **enp0s9 (Internal):** Toggle to **ON**. Click **Configure...** -> **IPv4 Settings** -> Method: **Manual**. Add your designated Private IP (e.g., `10.10.10.121`) and Netmask (`255.255.255.0`).
   * Click **Done**.
6. Click **Begin Installation**.



---

## Phase 5: User Setup and Finalization

While the installation progress bar runs, configure the following:

1. **Root Password:** Set your "super secret" password for the `root` user.
2. **User Creation:** Create an administrative user.
   * Full Name / User Name: `admin` (or your preferred name).
   * Check **Make this user administrator**.
   * Set your "super secret" password.
   * *Note: Do NOT manually create the `oracle` user here. It is highly recommended to let the `oracle-database-preinstall-19c` package create the `oracle` user and its associated groups automatically later.*
3. Once the installation is 100% complete, click **Reboot**.

Your RAC node is now provisioned and ready for Grid Infrastructure preparation!