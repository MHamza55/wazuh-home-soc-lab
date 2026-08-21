# Architecture & Design Decisions

## Overview

The lab models a minimal but complete SOC monitoring pipeline:

```
endpoint (agent)  ──▶  SIEM manager  ──▶  indexer  ──▶  dashboard (analyst)
   Ubuntu               Wazuh manager     OpenSearch     Wazuh dashboard
   10.0.5.5             ───────────── 10.0.5.3 ─────────────
```

All components run as VirtualBox VMs on a single Windows 11 host.

## Network design

| Decision | Rationale |
|---|---|
| **NAT Network** (`lab-net`, 10.0.5.0/24) rather than Bridged | Isolates lab traffic from the home network, and gives consistent addressing regardless of the host's physical network (Wi-Fi, campus, etc.). Attack traffic never touches the real LAN. |
| **Port forwarding** to `127.0.0.1` | The NAT network isn't reachable from the host directly, so only the specific ports needed (dashboard 8443, SSH 2222/2223) are exposed to loopback — nothing is published to the wider network. |
| Single **all-in-one** Wazuh appliance | Manager, indexer, and dashboard on one VM keeps the lab lightweight while still exercising the full detection pipeline. A production deployment would separate the indexer for scale. |

## Component sizing

| VM | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Wazuh server | 4 | 8 GB | 25 GB | Indexer is memory-hungry; 8 GB is the practical minimum. |
| Ubuntu target | 2 | 2 GB | 20 GB | Lightweight monitored endpoint. |

## Data flow

1. `sshd` (and other services) on the endpoint write to system logs.
2. The **Wazuh agent** reads those logs and forwards events to the manager
   (`1514/udp`).
3. The **manager** decodes and evaluates events against its ruleset, generating
   alerts and enriching them with MITRE ATT&CK and compliance mappings.
4. Alerts are stored in the **indexer** and visualised in the **dashboard**.

## Security posture of the lab itself

- Default indexer/dashboard/API credentials rotated (see `../TROUBLESHOOTING.md`).
- Lab network isolated from the home LAN.
- No credentials committed to version control.
- Clean-state VM snapshots taken so the environment is disposable and reproducible.

## Known limitations / future improvements

- DHCP addressing can drift across reboots — a static IP for the server is planned.
- Single-node indexer (no clustering) — fine for a lab, not production-representative.
- Only one endpoint OS so far — a Windows + Sysmon endpoint is the next phase.
