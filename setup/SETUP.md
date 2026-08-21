# Lab Setup Guide

A reproducible build of the home SOC lab from scratch. Tested on Windows 11 with
Oracle VirtualBox 7.2.14 and Wazuh 4.14.6.

## Contents
1. [Host and software](#1-host-and-software)
2. [Isolated lab network](#2-isolated-lab-network)
3. [Wazuh server (OVA)](#3-wazuh-server-ova)
4. [Access — port forwarding](#4-access--port-forwarding)
5. [Server hardening](#5-server-hardening)
6. [Ubuntu target endpoint](#6-ubuntu-target-endpoint)
7. [Agent enrollment](#7-agent-enrollment)
8. [Verification](#8-verification)

---

## 1. Host and software

| Item | Value |
|---|---|
| Hypervisor | Oracle VirtualBox 7.2.14 + Extension Pack |
| SIEM | Wazuh 4.14.6 OVA (all-in-one appliance) |
| Endpoint | Ubuntu Server 26.04 LTS |

Verify the Wazuh OVA download before importing:

```powershell
Get-FileHash .\wazuh-4.14.6.ova -Algorithm SHA512
# compare against the published .sha512 checksum
```

---

## 2. Isolated lab network

A dedicated NAT network keeps lab traffic isolated from the home network while
still allowing outbound internet for package installs. It also gives predictable
addressing regardless of the host's physical network.

VirtualBox → **Tools → Network → NAT Networks → Create**:

| Setting | Value |
|---|---|
| Name | `lab-net` |
| IPv4 prefix | `10.0.5.0/24` |
| DHCP | Enabled |
| IPv6 | Disabled |

CLI equivalent:

```powershell
VBoxManage natnetwork add --netname lab-net --network "10.0.5.0/24" --enable --dhcp on
```

Every VM in the lab attaches Adapter 1 to **NAT Network → lab-net**.

---

## 3. Wazuh server (OVA)

1. **File → Import Appliance** → select `wazuh-4.14.6.ova`.
2. Resources: **4 vCPU**, **8192 MB RAM**. Tick *Reinitialize the MAC address of all network cards*.
3. Settings → Network → Adapter 1 → **NAT Network → lab-net**.
4. Start, log in at the console (`wazuh-user` / `wazuh`), find the IP:

```bash
ip a        # server came up on 10.0.5.3
```

---

## 4. Access — port forwarding

The NAT network isn't directly reachable from the host, so map host loopback ports
to the guests. VirtualBox → NAT Networks → `lab-net` → **Port Forwarding**:

| Name | Protocol | Host IP | Host Port | Guest IP | Guest Port |
|---|---|---|---|---|---|
| wazuh-dash | TCP | 127.0.0.1 | 8443 | 10.0.5.3 | 443 |
| wazuh-ssh | TCP | 127.0.0.1 | 2222 | 10.0.5.3 | 22 |
| ubuntu-ssh | TCP | 127.0.0.1 | 2223 | 10.0.5.5 | 22 |

- Dashboard: `https://127.0.0.1:8443` (self-signed cert warning is expected)
- Server shell: `ssh wazuh-user@127.0.0.1 -p 2222`
- Endpoint shell: `ssh <user>@127.0.0.1 -p 2223`

---

## 5. Server hardening

**Rotate all default indexer/dashboard passwords:**

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -a -A
# save the generated passwords in a password manager
sudo systemctl restart wazuh-indexer wazuh-dashboard filebeat wazuh-manager
```

**Raise the manager start timeout** (the appliance can exceed systemd's default 45s
and fail to fully start — see TROUBLESHOOTING):

```bash
sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d
printf '[Service]\nTimeoutStartSec=600\n' | sudo tee /etc/systemd/system/wazuh-manager.service.d/override.conf
sudo systemctl daemon-reload
```

**Snapshot** the clean, hardened server in VirtualBox (`clean-install`) before going further.

---

## 6. Ubuntu target endpoint

1. New VM: `ubuntu-target`, Linux/Ubuntu 64-bit, **2 vCPU**, **2048 MB RAM**, **20 GB** disk.
2. Adapter 1 → **NAT Network → lab-net**.
3. Install Ubuntu Server with defaults, **and tick "Install OpenSSH server"** (required for the SSH detection scenario).
4. After reboot:

```bash
ip a        # endpoint came up on 10.0.5.5
ping -c 3 10.0.5.3   # confirm it reaches the Wazuh server
```

---

## 7. Agent enrollment

In the dashboard: **Agents → Deploy new agent** → package **DEB amd64**,
server address `10.0.5.3`, agent name `ubuntu-target`. Run the generated command
on the endpoint, then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

If you need to (re)enroll manually with a clean name:

```bash
sudo systemctl stop wazuh-agent
sudo /var/ossec/bin/agent-auth -m 10.0.5.3 -A ubuntu-target
sudo systemctl start wazuh-agent
```

---

## 8. Verification

- Dashboard → **Endpoints** shows `ubuntu-target` as **active**.
- The agent reports OS, cores, and memory in **System inventory**.

The lab is now ready for the detection scenarios in [`../detections/`](../detections/).

---

## Running the lab headless

Start/stop the VMs without console windows:

```powershell
cd "C:\Program Files\Oracle\VirtualBox"
.\VBoxManage.exe startvm "Wazuh v4.14.6 OVA" --type headless
.\VBoxManage.exe startvm "ubuntu-target" --type headless
.\VBoxManage.exe list runningvms

# graceful shutdown
.\VBoxManage.exe controlvm "ubuntu-target" acpipowerbutton
.\VBoxManage.exe controlvm "Wazuh v4.14.6 OVA" acpipowerbutton
```
