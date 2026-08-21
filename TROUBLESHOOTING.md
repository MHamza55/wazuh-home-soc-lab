# Troubleshooting Log

Real problems encountered while building the lab, and how each was resolved.
This log is deliberately kept — diagnosing and fixing these is where most of the
actual learning happened.

---

## 1. Wazuh manager fails to start — `systemd` timeout

**Symptom:** `Job for wazuh-manager.service failed because a timeout was exceeded.`
`wazuh-control status` showed `analysisd`, `remoted`, `db` running but
`modulesd`, `logcollector`, `monitord` stopped — a *partial* startup.

**Cause:** the all-in-one appliance takes longer than systemd's default 45-second
start timeout, so systemd killed the startup partway through.

**Fix:** a full control-script restart brought all daemons up cleanly, and a
systemd drop-in raised the timeout permanently:

```bash
sudo /var/ossec/bin/wazuh-control restart
sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d
printf '[Service]\nTimeoutStartSec=600\n' | sudo tee /etc/systemd/system/wazuh-manager.service.d/override.conf
sudo systemctl daemon-reload
# verify: systemctl show wazuh-manager -p TimeoutStartUSec  ->  10min
```

> Note: on a single-node install, `clusterd`, `maild`, `agentlessd`, `integratord`,
> `dbd`, and `csyslogd` are **expected** to be stopped — they aren't configured.

---

## 2. Dashboard: "Something went wrong / timeout of 20000ms exceeded"

**Symptom:** dashboard loaded but threw a timeout error after the VM had just booted.

**Cause:** the Wazuh indexer and API take a couple of minutes to become ready after
boot; the dashboard's request to the API timed out in the meantime. Also worsened
by host CPU contention when both VMs ran at once.

**Fix:** wait for services to settle and refresh. Health can be confirmed directly:

```bash
sudo /var/ossec/bin/wazuh-control status
curl -k -u admin https://127.0.0.1:9200/_cluster/health?pretty   # indexer
```

---

## 3. Default credentials — partial rotation

**Finding:** the password tool rotated the indexer/dashboard users but printed
`Wazuh API admin credentials not provided, Wazuh API passwords not changed` —
the API kept its default `wazuh:wazuh`.

**Fix / hardening:**

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -a -A
sudo systemctl restart wazuh-indexer wazuh-dashboard filebeat wazuh-manager
```

Documenting "identified and remediated default credentials" is itself a valid
security finding.

---

## 4. Attack from Wazuh server blocked by FIPS mode

**Symptom:** running the SSH brute-force loop from the Wazuh server failed
immediately with:
`kex_gen_client: Key exchange type c25519 is not allowed in FIPS mode`.

**Cause:** the Wazuh appliance's SSH client runs in FIPS mode and refuses the
modern curve25519 key exchange, so the connection never reached the target — no
authentication attempt was generated.

**Fix:** launch the attack from a non-FIPS host (the Ubuntu endpoint). Detections
appeared immediately afterward.

---

## 5. Agent name typo

**Symptom:** agent enrolled as `ubuntu-targer` instead of `ubuntu-target`.

**Cause:** typo at enrollment time; the name propagates everywhere.

**Fix:** re-enroll with the correct name and remove the stale entry:

```bash
# on the endpoint
sudo systemctl stop wazuh-agent
sudo /var/ossec/bin/agent-auth -m 10.0.5.3 -A ubuntu-target
sudo systemctl start wazuh-agent

# on the manager
sudo /var/ossec/bin/manage_agents   # (R)emove the old ID, then (Q)uit
```

The agent reappeared cleanly as `ubuntu-target` (ID 002).

---

## 6. DHCP address drift

**Observation:** both VMs use DHCP on `lab-net`, so addresses can change across
reboots. If the dashboard or SSH forwards stop working, re-check with `ip a` and
update the port-forwarding **Guest IP** values accordingly.

**Improvement (planned):** assign the Wazuh server a static address so the lab is
fully reproducible without editing forwards.

---

## 7. VirtualBox soft lockups (`watchdog: BUG: soft lockup`)

**Symptom:** during Ubuntu install, `CPU stuck for 23s` watchdog messages.

**Cause:** host CPU/RAM over-commitment — running the 8 GB / 4 vCPU Wazuh VM and
the Ubuntu VM simultaneously on a constrained host.

**Mitigations:** don't run both VMs during heavy installs; keep only what's needed
running; on Windows, disabling Hyper-V-based features (Core Isolation / Memory
Integrity, WSL2, Docker Desktop) removes VirtualBox's degraded execution mode.
