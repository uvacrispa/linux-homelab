# Linux Homelab

A lightweight local infrastructure designed for practicing Linux systems administration, network diagnostics, infrastructure automation (Ansible), and telemetry monitoring (Prometheus & Grafana). Hosted entirely on macOS using the UTM hypervisor.

---

## Network Architecture

The homelab consists of two isolated Ubuntu Server virtual machines connected through a virtualized **Shared Network (NAT)** provided by macOS and UTM. This setup ensures seamless VM-to-VM and Host-to-VM communication, independent of the host's physical Wi-Fi or local router.

```text
               +----------------------------------------+
               |            macOS Host (Mac)            |
               |        ~/.ssh/config (`vm1`/`vm2`)     |
               +-------------------+--------------------+
                                   |
                     Shared Network (NAT: 192.168.65.0/24)
                                   |
         +-------------------------+-------------------------+
         |                                                   |
+--------v-------------------+              +----------------v-----------+
| VM 1: `tele`               |              | VM 2: `ubuntuserver`       |
| OS: Ubuntu Server (ARM64)  | <==========> | OS: Ubuntu Server (ARM64)  |
| IP: 192.168.65.6           |   ICMP/SSH   | IP: 192.168.65.5           |
| UFW: Deny In / Allow 22/tcp|              | UFW: Deny In / Allow 22/tcp|
+----------------------------+              +----------------------------+
```

### Node Specifications

| Node | Hostname | Role / Purpose | IP Address | Subnet Mask | Specs |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Node 1** | `tele` | Target Application Server | `192.168.65.6` | `255.255.255.0` (`/24`) | 2 vCPU, 2 GB RAM |
| **Node 2** | `ubuntuserver` | Infrastructure & Backup Server | `192.168.65.5` | `255.255.255.0` (`/24`) | 2 vCPU, 2 GB RAM |

---

## Security & Server Hardening

Both virtual machines have been configured following baseline security practices:

### 1. Passwordless SSH Key Authentication
- An **Ed25519** elliptic-curve keypair was generated on the macOS host (`ssh-keygen -t ed25519`).
- Public keys were deployed to each node using `ssh-copy-id`.
- Password authentication was explicitly disabled in the SSH daemon configuration (`/etc/ssh/sshd_config`):
  ```ini
  PasswordAuthentication no
  PubkeyAuthentication yes
  ```
- **Impact:** Eliminates the risk of SSH brute-force and dictionary attacks on port 22.

### 2. Uncomplicated Firewall (UFW)
- Applied a strict **least-privilege / whitelist** network policy:
  - Default Incoming: **DENY** (all unsolicited incoming packets are dropped).
  - Default Outgoing: **ALLOW** (instances can freely fetch packages, query DNS, and reach the internet).
- Explicitly whitelisted management port 22 before enabling the firewall:
  ```bash
  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  sudo ufw allow 22/tcp comment 'SSH Administration'
  sudo ufw enable
  ```
- Current firewall status (`sudo ufw status verbose`):
  ```text
  Status: active
  Logging: on (low)
  Default: deny (incoming), allow (outgoing), disabled (routed)
  New profiles: skip

  To                         Action      From
  --                         ------      ----
  22/tcp                     ALLOW IN    Anywhere
  22/tcp (v6)                ALLOW IN    Anywhere (v6)
  ```

---

## Host SSH Configuration

To simplify operations and remove the need to memorize raw IP addresses, the macOS host's `~/.ssh/config` is configured as follows:

```ssh-config
Host vm1
    HostName 192.168.65.6
    User tele
    IdentityFile ~/.ssh/id_ed25519

Host vm2
    HostName 192.168.65.5
    User tele
    IdentityFile ~/.ssh/id_ed25519
```

Connecting to either node requires just:
```bash
ssh vm1
# or
ssh vm2
```

---

## Validation & Health Checks

Connectivity and isolation were verified with the following manual test suite:
- [x] **Outbound connectivity & DNS resolution:** Successful packet transmission (`ping -c 3 1.1.1.1` and package updates via `apt`).
- [x] **Internal East-West connectivity:** Bidirectional ICMP and TCP connectivity between `192.168.65.6` and `192.168.65.5`.
- [x] **Host-to-Guest connectivity:** Seamless key-based authentication from macOS host terminal.
- [x] **Firewall enforcement:** UFW active with default deny on unauthorized ports.

---
