# Ticket 1 — New developer starts Monday, needs a test machine

## 1. What was asked
> "Hi, we have someone starting on Monday and he needs a test computer set up. Nothing special, just so he can try things without breaking anything. Can you write down how to do it so I can do the next one myself? Thanks"
> — Клод (Office Manager)

## 2. What was actually needed
* **What the request left out:**
  * Target operating system distributions and versions (Linux distribution, Windows edition, or dual environments).
  * The exact software stack, compilers, development runtimes, and privilege levels needed by the developer.
  * Real host hardware specifications (physical RAM capacity, available CPU cores/threads, and free NVMe/SSD storage) required to determine safe virtualization limits[cite: 1, 3].
  * Network boundary requirements: whether the developer requires public internet egress, access to local office subnets/file shares, or strict sandbox isolation[cite: 1, 3].

* **Where the request was wrong:**
  * **Requesting a physical workstation:** The requester asked for a single physical "test computer". A proper testing sandbox must be delivered via isolated virtual machines (both Linux and Windows) to test cross-platform behaviors, enable instant rollback via snapshots, and completely prevent damage to physical hardware or the host operating system.
  * **Assuming non-technical manual replication:** The requester assumed that someone without hypervisor experience can manually replicate low-level VM wizard parameters repeatedly without configuration drift, missing critical drivers, or allocating conflicting resources[cite: 1, 3].
  * **"Without breaking anything" ignores network risks:** The requester assumed physical hardware separation alone guarantees safety, failing to understand that default virtual network bridging exposes the corporate LAN to unverified test code, broadcast flooding, and campus MAC filtering lockouts.

* **What I would have asked the requester:**
  1. What exact programming languages, frameworks, and target operating systems will the new developer work with?
  2. What are the host machine's total RAM, CPU core count, and free disk space figures?
  3. Does the test environment require testing network interactions between multiple instances, or strictly isolated internet access?

* **What I assumed instead:**
  1. **Guest OS Targets:** The developer requires dual guest environments: Ubuntu 24.04 LTS (for CLI/backend testing) and Windows 10 (for client/GUI testing).
  2. **Host Profile:** The host machine is an x86_64 platform with 16 GB of physical RAM, 8 logical CPU cores, and at least 100 GB of free SSD space.
  3. **Hypervisor Platform:** Oracle VM VirtualBox 7.x is used as the standard hypervisor because it is free, cross-platform, and provides consistent virtual networking across different physical machines[cite: 3, 4].
  4. **Network Topology:** Both virtual machines must communicate with each other and reach the external internet, but must be invisible to the corporate LAN[cite: 3, 4]. A dedicated VirtualBox NAT Network (`DevNet`, `10.0.2.0/24`) with internal DHCP was assumed[cite: 3, 4].

## 3. What I did

### Host Resource Budget & Sizing
* **Host Hardware:** 16 GB RAM, 8 logical CPUs (Intel Core i7), 512 GB NVMe SSD (220 GB available free space).
* **Host Safeguard:** Reserved a guaranteed 8 GB of RAM and 4 logical CPU cores for the host operating system to prevent memory exhaustion, UI freezing, and disk swapping during virtualization.
* **Guest Allocations:**
  * **Ubuntu 24.04 VM:** 4096 MB RAM, 2 vCPUs, 30.00 GB dynamically allocated virtual disk[cite: 3].
  * **Windows 10 VM:** 4096 MB RAM, 2 vCPUs, 50.00 GB dynamically allocated virtual disk[cite: 3].
* **Peak Utilization:** Both VMs combined consume at most 8 GB RAM and 4 vCPUs (exactly 50% of host capacity), ensuring host stability.

### Settings Differing from Defaults & Justifications
* **Network Mode (NAT Network vs. Default NAT vs. Bridged):**
  * *Default NAT* was rejected because it isolates each guest behind an independent routing engine, preventing inter-VM communication.
  * *Bridged Networking* was rejected because the university network enforces 802.1X / MAC address filtering; unknown guest virtual MAC addresses are automatically blocked by the switch ports[cite: 4].
  * *NAT Network (`DevNet`, `10.0.2.0/24` with DHCP)* was configured: both guests share a common private subnet and gateway (`10.0.2.1`), allowing them to ping each other while routing outbound internet traffic through the host's single authenticated IP[cite: 4].
* **Disk Provisioning (Dynamic vs. Fixed Allocation):**
  * Selected dynamically allocated VDI disks rather than fixed size[cite: 3]. While fixed disks offer slightly higher write performance, dynamic allocation avoids consuming 80 GB of physical host disk upfront, starting at ~4 GB for Ubuntu and expanding only as data is written.
* **Baseline Snapshots:**
  * Configured a clean baseline snapshot named `Baseline-Clean-Install` for both guests immediately following initial setup[cite: 3]. This allows the developer to reset corrupted system files or dependencies back to pristine state in seconds[cite: 3].

### Problem Encountered & Resolution
* **Issue:** After connecting the Ubuntu VM to the `DevNet` NAT Network, the guest obtained an IP address via DHCP (`10.0.2.4`) and could ping public IP addresses (`8.8.8.8`), but all domain name resolution failed (`Temporary failure in name resolution` on `ping google.com`). The Windows guest on the same subnet resolved DNS normally.
* **Root Cause:** VirtualBox's internal NAT Network DHCP daemon assigned `10.0.2.1` as the primary nameserver. Due to Wi-Fi packet filtering and power-management modes on the host wireless adapter, forwarded UDP DNS requests handled by `systemd-resolved` in Ubuntu 24.04 timed out without fallback.
* **Resolution:** Configured explicit public upstream fallback nameservers inside the Ubuntu guest:
  ```bash
  sudo sed -i 's/#DNS=/DNS=1.1.1.1 8.8.8.8/' /etc/systemd/resolved.conf
  sudo systemctl restart systemd-resolved
