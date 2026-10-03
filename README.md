# myfirstrproject
# SOC Homelab — Wazuh SIEM

A single-node Wazuh SIEM running on my laptop, used to practise detecting and investigating attacks the way a Tier 1 SOC analyst would.

## Environment

| Component | Detail |
|---|---|
| Hardware | Acer Nitro ANV15-51, 16 GB RAM |
| OS | Ubuntu 24.04.4 LTS (dual boot with Windows 11) |
| SIEM | Wazuh 4.14.8, all-in-one install (indexer, manager, dashboard) |
| Firewall | `ufw` enabled, incoming connections denied |

## Labs

| # | Lab | Technique (MITRE ATT&CK) | Status |
|---|---|---|---|
| 1 | [SSH brute-force detection](#lab-1--ssh-brute-force-detection) | T1110.001 Password Guessing | Done |

---

## Lab 1 — SSH brute-force detection

### Goal

Simulate repeated failed SSH logins against the host and confirm that Wazuh detects the individual attempts and correlates them into a brute-force alert.

### Steps

1. Installed the SSH server and a tool to script password attempts:

   ```bash
   sudo apt install -y openssh-server sshpass
   ```

2. Ran 10 login attempts with a user that does not exist:

   ```bash
   for i in $(seq 1 10); do
     sshpass -p 'wrongpassword' ssh -o StrictHostKeyChecking=no hacker@localhost
   done
   ```

   Every attempt returned `Permission denied`.

3. In the Wazuh dashboard, opened **Threat Hunting → Events** and filtered with:

   ```
   rule.groups: sshd
   ```

### Results

The 10 attempts produced 20 alerts on agent `wynn-Nitro-ANV15-51`.

| Rule ID | Level | Description |
|---|---|---|
| 5710 | 5 | sshd: Attempt to login using a non-existent user |
| 5712 | 10 | sshd: brute force trying to get access to the system. Non existent user. |

![Threat Hunting events](images/threat-hunting-events.png)

Rule 5710 fires for each failed attempt. Rule 5712 is a correlation rule: Wazuh saw several 5710 events in a short window, grouped them, and raised the severity from 5 to 10.

Wazuh maps these alerts to MITRE ATT&CK **T1110.001 — Brute Force: Password Guessing**.

![MITRE ATT&CK T1110.001](images/mitre-t1110-001.png)

### What I learned

<!-- Rewrite these in your own words before publishing -->
- One attack produces many low-level alerts. The correlated alert (5712) is the one an analyst should act on.
- sshd writes two log lines per failed attempt for an invalid user, which is why 10 attempts became 20 alerts.
- MITRE ATT&CK IDs give a shared vocabulary for describing what an alert means.

### Possible next steps

- Add an active response so Wazuh blocks the source IP after rule 5712 fires.
- Repeat the attack with a real username and compare the rule IDs.

---

## Setup notes and troubleshooting

### First attempt: Wazuh in a VirtualBox VM on Windows 11

I first installed Wazuh in an Ubuntu Server 24.04 VM (6 GB RAM, 4 vCPU) under VirtualBox 7.2.20 on Windows 11. The install failed twice at the same point:

```
ERROR: wazuh-indexer could not be started.
```

**Diagnosis**

- `journalctl -u wazuh-indexer` showed no real error, only `start operation timed out`. The indexer was not crashing, it was too slow to start.
- The VM console showed kernel CPU stall warnings.
- The VirtualBox status bar reported **Execution Engine: native API** and **Nested Paging: Inactive**, which means VirtualBox was running on top of the Windows hypervisor instead of using VT-x directly.
- `msinfo32` showed **Virtualization-based security: Running** and "A hypervisor has been detected".

**What I tried**

- Turned off Memory Integrity.
- `bcdedit /set hypervisorlaunchtype off`
- Disabled the Virtual Machine Platform and Windows Hypervisor Platform features with `dism`.
- Set the Device Guard registry values to disable VBS.
- Reduced the VM to 2 vCPUs.

The hypervisor was still active after each reboot, and the indexer still timed out.

**Decision**

I stopped fighting the Windows configuration and installed Wazuh directly on the Ubuntu partition of the same laptop. On bare metal the indexer started in about 7 seconds and the full install finished in under 5 minutes.

**Takeaway**

Reading the service log before retrying saved time: a timeout pointed to performance, not to a broken install, which led to checking how the VM was being virtualised.

### Final install

```bash
sudo apt update && sudo apt upgrade -y
sudo ufw enable
curl -s -o wazuh-install.sh https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

Dashboard: `https://localhost`
