# README – Ubuntu Server Setup (Static IP with Netplan + Snapshot/Restore)

This document describes how to configure a **static IP address** on a new Ubuntu Server using **Netplan**, verify the configuration, and what to do **after creating/restoring a snapshot**.

---

## 1) Prerequisites

### 1.1 Switch to root user

```bash
sudo -i
```

> Purpose: You need root privileges to edit system network configuration.

---

## 2) Configure Static IP using Netplan

### 2.1 Identify the active Netplan YAML file

Netplan configuration files are stored in:

```bash
ls -la /etc/netplan/
```

Common filenames:

* `/etc/netplan/00-installer-config.yaml`
* `/etc/netplan/50-cloud-init.yaml`

> ⚠️ **Important note:**
> If `50-cloud-init.yaml` exists, it may be managed by **cloud-init**. Make sure you modify the correct file that is currently applied.

---

### 2.2 Edit the Netplan YAML file

Example:

```bash
nano /etc/netplan/00-installer-config.yaml
```

---

### 2.3 Change network adapter configuration from DHCP to Static

Find your network interface section (e.g. `ens33`, `ens160`, `eth0`, etc.)

Example (DHCP):

```yaml
dhcp4: true
```

Change to:

```yaml
dhcp4: false
```

---

### 2.4 Add static IP configuration

Add the following fields:

* `addresses`: Static IP with CIDR
* `gateway4`: Default gateway
* `nameservers`: DNS servers

✅ Example static IP config:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses: [192.168.1.110/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

**Explanation:**

* `192.168.1.110/24` = subnet mask `255.255.255.0`
* `gateway4: 192.168.1.1` = router / gateway to the network
* DNS servers: `8.8.8.8` and `8.8.4.4` (Google DNS)

---

### 2.5 Save and exit

In nano:

* Press `Ctrl + X`
* Press `Y`
* Press `Enter`

---

### 2.6 Apply Netplan configuration

```bash
netplan apply
```

> ⚠️ If you are connected via SSH and you lose network connectivity, it usually means the IP/gateway/interface config is incorrect. In that case, use console access to fix the YAML.

---

### 2.7 Verify the new IP address

```bash
ip a
```

Confirm the interface now has `192.168.1.110/24`.

Optional connectivity checks:

```bash
ip r
ping -c 3 8.8.8.8
ping -c 3 google.com
```

---

## 3) Create a Snapshot (after Static IP works)

Once the static IP configuration is confirmed working:

* Create a snapshot from your VM/cloud/hypervisor platform

Recommended snapshot naming:

* `ubuntu-base-static-ip-192.168.1.110`

---

# 4) After Snapshot Restore (Required Post-Restore Steps)

After restoring a snapshot (especially on a different VM/host), you may face:

* Network interface name changes
* Netplan not applying as expected
* Incorrect hostname/IP mappings in `/etc/hosts`

Follow this checklist.

---

## 4.1 Check Netplan configuration files

```bash
ls -la /etc/netplan/
cat /etc/netplan/*.yaml
```

If the network interface name changed (e.g. `ens33` → `ens160`), update the YAML accordingly.

Find interface name:

```bash
ip a
```

---

## 4.2 Apply Netplan again

```bash
netplan apply
```

Optional safe test (useful in remote sessions):

```bash
netplan try
```

---

## 4.3 Check and update `/etc/hosts`

Edit the hosts file:

```bash
nano /etc/hosts
```

Recommended format:

```text
127.0.0.1 localhost
127.0.1.1 <hostname>

192.168.1.110 <hostname>
```

> If the restored VM uses a different static IP, update the IP in `/etc/hosts`.

---

## 4.4 Reboot the server

```bash
reboot
```

---

## 5) Quick Troubleshooting

### 5.1 Validate YAML syntax / Netplan errors

If Netplan fails due to syntax/indentation issues:

```bash
netplan generate
```

This will print exact error lines.

---

### 5.2 No network after apply

Most common reasons:

* Incorrect interface name in YAML
* Wrong gateway IP
* Wrong subnet/CIDR mask

Check:

```bash
ip a
ip r
```

---

## 6) Recommended Full Netplan Example

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses: [192.168.1.110/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

---

✅ DONE

* Static IP configured successfully
* Netplan applied and verified
* Snapshot created and restore checklist prepared

