# Troubleshooting Log

Real problems I hit while building the lab, and how I fixed each one. I've kept
this deliberately — working through these is where most of the actual learning
happened, and honestly it's the part I'm most glad I documented.

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

First time I hit this I thought I'd wrecked the whole install. What threw me was
that most of the services were actually running fine; systemd had just given up
waiting before everything finished starting.

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

Honestly this one had me stuck for a bit. Nothing was happening at all, and I kept
assuming my rule or the agent was broken, until I actually read the SSH error and
spotted the FIPS line. Running the attack from the Ubuntu box instead sorted it
straight away.

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

---

## 8. Custom rule loaded but never fired

**Symptom:** after writing custom rule 100010 and confirming it compiled
(`wazuh-analysisd -t` clean, rule present in `local_rules.xml`), searching
`rule.id:100010` in Threat Hunting returned nothing.

**Cause:** two separate things, and I chased the wrong one first.
1. My first version of the rule used `<if_sid>550,554</if_sid>` with a `pcre2`
   field pattern, which didn't match the file path. I simplified it to
   `<if_group>syscheck</if_group>` with a plain substring match on `authorized_keys`.
2. The bigger one: the endpoint had re-registered as a new agent ID, and I'd been
   filtering Threat Hunting on the old, now-disconnected agent the whole time. The
   alerts were firing fine, I just wasn't looking at the right agent.

**Fix:** simplify the rule's matching, remove the stale agent filter (and clean up
the duplicate agent so only one `ubuntu-target` remained), then re-trigger.

**Lesson:** "no results" in the dashboard doesn't mean the detection failed. Confirm
the rule compiled, confirm the event is arriving, and check you're looking at the
right agent and time range before assuming the rule is wrong. This was easily the
most useful debugging lesson of the whole build.
