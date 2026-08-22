# Detection Write-up: SSH Brute Force

**Scenario:** an attacker attempts to guess valid SSH credentials on a Linux
endpoint by repeatedly authenticating with wrong passwords. This is one of the
most common attacks against internet-facing Linux hosts.

|||
|-|-|
|**Target**|`ubuntu-target` — 10.0.5.5 (Ubuntu Server 26.04, agent ID 002)|
|**Technique**|MITRE ATT\&CK **T1110 — Brute Force** (Credential Access)|
|**Wazuh rules**|**5710** — attempt to log in with a non-existent user · **5503** — PAM login failed · **2502** — repeated password failures (escalated, level 10)|
|**Compliance**|PCI DSS **10.2.4**, **10.2.5**|

\---

## 1\. Hypothesis

If an attacker repeatedly fails SSH authentication against the endpoint, the
endpoint's `sshd` writes "authentication failure" lines to `/var/log/auth.log`.
The Wazuh agent forwards these to the manager, which should:

1. raise an alert per failed attempt (rule 5710 / 5503), and
2. raise a **correlated** higher-severity alert once several failures occur in a
short window (rule 2502, level 10), because a burst of failures is the
signature of brute forcing rather than a user fat-fingering a password.

\---

## 2\. Attack execution

Run from the Ubuntu endpoint against its own SSH service. The options force a
password prompt (rather than key-based auth) so that each attempt produces a
genuine authentication failure:

```bash
for i in $(seq 1 8); do
  ssh -o StrictHostKeyChecking=no \\\\
      -o PreferredAuthentications=password \\\\
      -o PubkeyAuthentication=no \\\\
      baduser@10.0.5.5
done
# enter any wrong password at each prompt
```

Fully automated variant (requires `sshpass`):

```bash
sudo apt install -y sshpass
for i in $(seq 1 8); do
  sshpass -p wrongpass ssh -o StrictHostKeyChecking=no \\\\
      -o PreferredAuthentications=password \\\\
      -o PubkeyAuthentication=no \\\\
      baduser@10.0.5.5
done
```

> \\\*\\\*Lab note:\\\*\\\* the same command run \\\*from the Wazuh server\\\* fails with
> `Key exchange type c25519 is not allowed in FIPS mode` — the appliance's SSH
> client runs in FIPS mode. The attack must be launched from a non-FIPS host
> (the Ubuntu box). See `../TROUBLESHOOTING.md`.

\---

## 3\. What Wazuh detected

The Wazuh **decoders** parse each `sshd` failure line, and the **rules** classify
them:

|Rule ID|Level|Meaning|
|-|-|-|
|5710|5|`sshd: Attempt to login using a non-existent user` — one per attempt|
|5503|5|`PAM: User login failed` — the PAM stack rejecting the auth|
|2502|10|`syslog: User missed the password more than one time` — the escalated alert that fires once repeated failures accumulate|

The jump from level 5 (individual failure) to level 10 (repeated-failure /
brute-force alert) is the core value of correlation: a single failed login is
noise; eight in a few seconds is an attack.

**Why these rule IDs and not 5760/5712?** The attack used `baduser`, an account
that does not exist, so `sshd` raises the *non-existent user* rule **5710** rather
than the *failed password for a valid user* rule **5760**. Had a real username
been used with a wrong password, 5760 (and its correlation rule 5712) would have
fired instead. Recognising which rule fires and why is part of the analysis.

**Agent dashboard** (`Endpoints → ubuntu-target`) attributed the activity to:

* **MITRE ATT\&CK → Credential Access** (T1110) as the top tactic
* **PCI DSS 10.2.4** (invalid logical access attempts) and **10.2.5**
(use of identification and authentication mechanisms)

### Evidence

|Screenshot|Shows|
|-|-|
|`../screenshots/01-dashboard-overview.png`|Agent active + MITRE ATT\&CK (Credential Access) + PCI DSS classification after the attack|
|`../screenshots/02-threat-hunting-events.png`|Individual events in Threat Hunting — rules 5710 / 5503, and the level-10 rule 2502|
|`../screenshots/03-alert-detail.png`|Expanded event document showing decoder, source IP, source user, and full log|

\---

## 4\. Triage — how an analyst would read this

1. **Signal vs noise:** the level-10 alert (rule 2502) is the one to action; the
level-5 failures (5710 / 5503) are supporting evidence.
2. **Source:** in a real environment, check the source IP of the attempts. A
single external IP hammering one account → brute force. One IP against many
accounts → password spraying (a different technique, T1110.003).
3. **Outcome:** confirm whether any attempt *succeeded* (rule 5715, "sshd
authentication success") immediately after the failures — that would indicate
a compromise, not just an attempt.

\---

## 5\. Defensive recommendations

Real-world mitigations this detection would drive:

* Enforce **key-based authentication** and set `PasswordAuthentication no` in `sshd\\\_config`.
* Deploy **fail2ban** or Wazuh **active response** to auto-block the source IP after N failures.
* Restrict SSH exposure with firewall rules / a bastion (jump) host.
* Require **MFA** for remote administrative access.

\---

## 6\. Next iteration

* Add **active response** so Wazuh automatically firewalls the attacker after the
level-10 2502 alert, and document the block firing.
* Repeat against the (planned) Windows endpoint to compare Linux vs Windows
authentication telemetry (event ID 4625).

