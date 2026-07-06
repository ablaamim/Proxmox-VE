# Proxmox VE Production Installation Guide

This a complete checklist for deploying Proxmox VE on bare metal servers in a production environment.

---

## 1. Pre-Installation Planning

Choice of PVE version : Proxmox VE 9.2 

### 1.1 Network Planning
Keep it simple : a single NIC with one bridge (`vmbr0`) covering both management and VM traffic is enough for most deployments.

- Reserve a **static IP** for the node (do NOT use DHCP for the host)
- Plan the hostname/FQDN and ensure forward + reverse DNS resolution works correctly (important for SSL certificates and, later, clustering if you add nodes)
- Note down the subnet and gateway before installing

### 1.2 Storage Strategy
Decide the filesystem/storage backend up front, this is hard to change post-install:

- **ZFS (RAIDZ0)**: Best for data integrity, snapshots, replication. 

### 1.3 Licensing
- Decide between the **Enterprise Repository** (paid subscription, stable/tested updates) and **No-Subscription Repository** (free, community, less tested).
- Production deployments should purchase at least a **Community or Basic subscription** for support and enterprise repo access.

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

---

## 2. BIOS/UEFI Configuration

- Enable **VT-x/VT-d** (Intel) or **AMD-V/AMD-Vi** (AMD) for virtualization and IOMMU passthrough
- Enable **IOMMU** if planning PCIe passthrough (GPUs, NICs, HBAs)
- Disable unused peripherals (unused SATA ports, serial, etc.)
- Set boot mode consistently (UEFI recommended over Legacy BIOS)
- Configure RAID controller in **HBA/IT mode** if using ZFS (avoid hardware RAID abstraction)
- Enable **ECC memory reporting**
- Update to the latest stable BIOS/firmware version before install

---

## 3. Installation Steps

1. **Download** the latest Proxmox VE ISO from the official site and verify its checksum/GPG signature.
2. **Create bootable USB** using `dd`, Rufus, Etcher, or Ventoy.
3. **Boot the target server** from the USB and select "Install Proxmox VE".
4. **Accept EULA**, then select the **target disk(s)**:
   - Click "Options" to choose filesystem (ZFS, LVM, BTRFS)
   - For ZFS, select RAID level (Mirror, RAIDZ1, RAIDZ2) and target disks
5. **Configure location/timezone and keyboard layout**.
6. **Set root password** (use a strong password; a password manager-generated one is recommended) and **admin email** (used for update/health notifications).
7. **Configure network**:
   - Select the management NIC
   - Set static **hostname (FQDN)**, **IP/CIDR**, **gateway**, and **DNS server**
8. **Review summary** and start installation.
9. **Reboot** and remove the installation media.
10. Access the **Web GUI** at `https://<node-ip>:8006` to confirm the install.

---

## 4. Post-Installation Configuration

### 4.1 Repository Configuration
```bash
# Disable the enterprise repo if you don't have a subscription
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list

# Add the no-subscription repo (or configure enterprise repo with your key if licensed)
echo "deb http://download.proxmox.com/debian/pve $(awk -F= '/VERSION_CODENAME/{print $2}' /etc/os-release) pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

# Disable the subscription nag (optional, community workaround)
apt update && apt full-upgrade -y
```

### 4.2 System Updates
```bash
apt update && apt full-upgrade -y
reboot
```
- Schedule regular update windows and test updates on a non-production node first if running a cluster.

### 4.3 Hostname & DNS Verification
```bash
hostname -f          # Confirm FQDN resolves correctly
cat /etc/hosts        # Ensure the node's IP + FQDN + short name are listed
```

### 4.4 Time Synchronization
```bash
systemctl status chrony    # or systemd-timesyncd, depending on version
```
- Accurate time is critical for cluster quorum (corosync) and certificate validation. Point to internal NTP servers if available.

### 4.5 Storage Configuration
- Configure additional storage via **Datacenter → Storage** (NFS, iSCSI, Ceph, ZFS pools, directory-based).
- Enable **email/webhook notifications** for storage/ZFS pool health (`zpool status` monitoring).
- Set up scheduled **ZFS scrubs**:

### 4.6 Networking
- The installer already creates a default bridge (`vmbr0`) on your management NIC — use this same bridge for VM traffic too, keeping things simple.
- Confirm the config in `/etc/network/interfaces` looks like:
```
auto vmbr0
iface vmbr0 inet static
    address <node-ip>/<cidr>
    gateway <gateway-ip>
    bridge-ports <physical-nic>
    bridge-stp off
    bridge-fd 0
```
- Apply any changes with `ifreload -a` (avoid a reboot when possible, but validate with out-of-band/IPMI access first in case connectivity drops).

---

## 5. Security Hardening

### 5.1 Firewall
- Enable the **Proxmox Firewall** at Datacenter level.
- Create rules restricting the Web GUI (port 8006) and SSH (port 22) to trusted management subnets only.
- Set default policy to `DROP` for input, allow only required ports (8006, 22, corosync 5405-5412, VNC/SPICE ranges as needed).

### 5.2 SSH Hardening
```bash
# /etc/ssh/sshd_config
PermitRootLogin prohibit-password   # Disable password login for root; use SSH keys
PasswordAuthentication no
```
- Deploy SSH keys for all administrative access; disable password authentication once keys are confirmed working.
- Consider changing the default SSH port and/or restricting via firewall rules.

### 5.3 Two-Factor Authentication
- Enable **TFA (TOTP or WebAuthn/U2F)** for all Proxmox GUI users under **Datacenter → Permissions → Two Factor**.

### 5.4 User & Role Management
- Avoid daily use of the `root@pam` account.
- Create named admin accounts with appropriate **roles/permissions** (principle of least privilege) via **Datacenter → Permissions**.
- Integrate with **LDAP/Active Directory** or **Realm-based authentication** for centralized identity management if applicable.

### 5.5 Certificates
- Replace the self-signed certificate with a valid one from an internal CA or Let's Encrypt (Proxmox has built-in ACME support under **Datacenter → ACME**).


## 11. Final Production Checklist

- [ ] BIOS/UEFI virtualization + IOMMU enabled
- [ ] Static IP, correct FQDN, forward/reverse DNS working
- [ ] Repository configured (enterprise or no-subscription) and system fully updated
- [ ] Storage backend configured with redundancy (ZFS/RAID/Ceph)
- [ ] Network bridge (`vmbr0`) configured with static IP and tested
- [ ] Firewall enabled with restrictive rules
- [ ] SSH key-based auth enforced, password auth disabled
- [ ] TFA enabled for all admin accounts
- [ ] Valid SSL certificate installed

---

*Reference: Official Proxmox VE Administration Guide — https://pve.proxmox.com/pve-docs/*