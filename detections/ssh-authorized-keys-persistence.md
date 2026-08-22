# Detection Write-up: SSH `authorized_keys` Persistence (Custom Rule)

**Scenario:** after gaining access to a Linux host, an attacker writes their own
public key into a user's `~/.ssh/authorized_keys` file. This gives them durable,
password-independent SSH access — they can log back in at any time, and rotating
the user's password does nothing to remove them. It is one of the most common
Linux persistence techniques.

This detection was **built from scratch**: File Integrity Monitoring was configured
to watch the `.ssh` directory in real time, and a **custom Wazuh rule** was authored
to escalate any change to `authorized_keys` into a high-severity, MITRE-tagged alert.

| | |
|---|---|
| **Target** | `ubuntu-target` — 10.0.5.5 (Ubuntu Server 26.04) |
| **Technique** | MITRE ATT&CK **T1098.004 — Account Manipulation: SSH Authorized Keys** |
| **Custom rule** | **100010** (level 12) |
| **Detection source** | Wazuh File Integrity Monitoring (syscheck), real-time |
| **Compliance** | PCI DSS **10.2.7** |

---

## 1. Detection design

Wazuh ships built-in FIM rules that fire a generic, low-severity alert (rule 550,
"Integrity checksum changed") whenever *any* monitored file changes. That is too
noisy and too low-signal to action on its own — a changed file could be anything.

The goal of this detection is to turn that generic file-change event into a
**specific, high-severity, attacker-contextualised alert** when the file that
changes is an `authorized_keys` file — because that specific change maps directly
to a known persistence technique.

Two pieces make it work:

1. **FIM configuration on the endpoint** — tell the agent to watch the `.ssh`
   directory in real time and record content changes.
2. **A custom correlation rule on the manager** — sit on top of the built-in FIM
   event, match only the `authorized_keys` path, raise the level to 12, and tag it
   with MITRE T1098.004.

---

## 2. Endpoint configuration (FIM)

On the monitored host, the following was added inside the `<syscheck>` block of
`/var/ossec/etc/ossec.conf`:

```xml
<directories realtime="yes" check_all="yes" report_changes="yes">/home/ubuntu/.ssh</directories>
```

- `realtime="yes"` — use inotify so changes are detected immediately, not on the
  next scheduled scan.
- `check_all="yes"` — track size, permissions, ownership, and content hashes.
- `report_changes="yes"` — record *what* changed inside the file (the added key),
  not just that it changed.

The agent was restarted to apply it:

```bash
sudo systemctl restart wazuh-agent
```

---

## 3. Custom detection rule (manager)

Authored in `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="local,syscheck,persistence,">
  <rule id="100010" level="12">
    <if_group>syscheck</if_group>
    <field name="file">authorized_keys</field>
    <description>SSH authorized_keys created or modified - possible persistence (MITRE T1098.004)</description>
    <mitre>
      <id>T1098.004</id>
    </mitre>
    <group>persistence,pci_dss_10.2.7,</group>
  </rule>
</group>
```

Line by line:

| Element | Purpose |
|---|---|
| `id="100010"` | Custom rule ID (the 100000+ range is reserved for user rules) |
| `level="12"` | High severity — surfaces in the "Level 12 or above" dashboards |
| `<if_group>syscheck</if_group>` | Only evaluate this rule for File Integrity Monitoring events |
| `<field name="file">authorized_keys</field>` | Match only when the changed file path contains `authorized_keys` |
| `<mitre><id>T1098.004</id>` | Tag the alert with the exact ATT&CK sub-technique |
| `<group>persistence,pci_dss_10.2.7,</group>` | Classify for the persistence tactic and PCI DSS reporting |

The ruleset was validated before reloading — this catches XML errors safely
without taking the manager down:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t     # must complete with no errors
sudo systemctl restart wazuh-manager
```

> **Engineering note:** the first version of this rule used
> `<if_sid>550,554</if_sid>` with a `pcre2` field pattern and did not match. It was
> simplified to `<if_group>syscheck</if_group>` with a plain substring match on
> `authorized_keys`, which is both more readable and more reliable. Debugging a
> rule that loads cleanly but never fires — by confirming it compiled, then
> checking the decoded field it was matching against — is a normal part of
> detection engineering.

---

## 4. Attack execution

Simulating the attacker dropping their key into the file:

```bash
echo 'ssh-ed25519 AAAAtest attacker@evil' >> ~/.ssh/authorized_keys
```

Real-time FIM detected the modification within seconds, the built-in syscheck rule
fired, and custom rule **100010** matched on top of it.

---

## 5. What Wazuh detected

| Field | Value |
|---|---|
| `rule.id` | 100010 |
| `rule.level` | 12 (high) |
| `rule.description` | SSH authorized_keys created or modified - possible persistence (MITRE T1098.004) |
| `rule.mitre.id` | T1098.004 |
| `syscheck.path` | /home/ubuntu/.ssh/authorized_keys |
| `rule.groups` | syscheck, persistence, pci_dss_10.2.7 |

The Threat Hunting dashboard classified the activity under the **Persistence**
tactic and the **SSH Authorized Keys** technique, with `report_changes` capturing
the exact key that was added.

### Evidence

| Screenshot | Shows |
|---|---|
| `../screenshots/04-custom-rule-authorized-keys.png` | Dashboard — level-12 alert count and the MITRE "SSH Authorized Keys" ring |
| `../screenshots/05-custom-rule-event-detail.png` | Events — the 100010 alerts under `ubuntu-target` with rule level and description |

---

## 6. Why this matters (analyst view)

- **High signal:** unlike a generic file-change alert, a level-12 hit on this rule
  means one specific, well-understood thing — someone changed an SSH key file, a
  common persistence and lateral-movement enabler.
- **Response:** confirm whether the added key is authorised. If not, remove it,
  rotate the account's credentials, hunt for how the attacker got write access in
  the first place, and check other users' `authorized_keys` for the same key.
- **Detection engineering value:** this demonstrates the full loop — configuring a
  telemetry source (FIM), authoring and validating a custom rule, mapping it to
  ATT&CK, and proving it fires against a realistic attack.

---

## 7. Hardening recommendations

- Set `~/.ssh` and `authorized_keys` to strict ownership/permissions and consider
  making `authorized_keys` immutable (`chattr +i`) where key rotation is managed
  centrally.
- Manage authorised keys through configuration management so any out-of-band change
  is, by definition, suspicious.
- Add **Wazuh active response** to alert or isolate the host automatically when
  rule 100010 fires.

---

## 8. Next iteration

- Extend the rule to every user's home directory (`/home/*/.ssh`) rather than a
  single user, and to `root`'s `.ssh`.
- Add a companion rule for new SSH key files being *created* and for changes to
  `/etc/ssh/sshd_config`.
