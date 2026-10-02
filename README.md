# SOC Home Lab: Endpoint Threat Detection with Wazuh, Sysmon & MITRE ATT&CK

## Overview
A hands-on Security Operations Center (SOC) home lab built to simulate 
real-world endpoint threat detection. The lab centers on a Wazuh SIEM 
deployment monitoring a Windows 7 endpoint instrumented with Sysmon, 
mapping detected activity to the MITRE ATT&CK framework.

## Architecture
- **Wazuh Manager**: Ubuntu Server VM (VirtualBox)
- **Monitored Endpoint**: Windows 7 VM ("PC2") running the Wazuh agent 
  and Sysmon (SwiftOnSecurity configuration)
- **Network**: Both VMs on a shared VirtualBox NAT Network
- **Access**: Wazuh dashboard reached via port forward to the host browser

## Setup Summary
1. Deployed Wazuh manager on Ubuntu Server
2. Installed Sysmon on the Windows 7 VM with the SwiftOnSecurity config
3. Installed and registered the Wazuh agent on the Windows 7 VM
4. Configured Wazuh to ingest Sysmon event logs
5. Simulated attacker behavior and validated detection and MITRE mapping

## The Sysmon Integration Fix
By default, the Wazuh agent does not read the Sysmon event channel. 
After installing Sysmon, no Sysmon-based alerts appeared in Wazuh despite 
the agent showing "Active." The fix was adding a localfile block to 
the agent's ossec.conf, pointing it at the Microsoft-Windows-Sysmon/Operational 
event channel with eventchannel log format. After restarting the Wazuh 
agent service, Sysmon-sourced events began appearing in the Wazuh 
dashboard and populating the MITRE ATT&CK view.

## Detections Achieved
Simulated attacker reconnaissance and privilege escalation activity on 
PC2 produced the following detections:

- Discovery commands (whoami, net user, systeminfo) → Rule 92031, Level 3, Discovery tactic
- New local user account created → Rule 60109/60110, Level 8, Persistence tactic
- User added to local Administrators group → Rule 60154, Level 12, Privilege Escalation tactic
- Windows logon activity → Rule 60106, Level 3, Initial Access tactic
- Registry key/value modification (FIM) → Rule 594/750, Level 5, Defense Evasion tactic

See screenshots/03-admin-group-detection.png for the level-12 
Administrators Group Changed alert, and screenshots/05-mitre-attack-pc2.png 
for the full MITRE ATT&CK breakdown scoped to PC2.

## Custom Detection Rule (In Progress)
To detect a classic "backdoor account" pattern — a new local user created 
and immediately escalated to Administrator — a custom correlation rule 
(ID 100100, Level 14) was written in local_rules.xml, correlating rule 
60154 against rule 60109 on the same TargetUserName field, mapped to 
MITRE techniques T1136.001 and T1078.003. Status: rule loads successfully 
and the manager runs clean, but end-to-end firing has not yet been 
confirmed. Next step is validating the field match against actual 
Sysmon/Windows event field names.

## Lessons Learned
- Wazuh's XML rule parser is strict — a single malformed tag can 
  silently break the whole rules file and prevent the manager from starting
- The timeframe tag in custom rules requires frequency to be meaningful; 
  it cannot be used standalone with if_matched_sid
- Sysmon integration with Wazuh is not automatic — the eventchannel 
  must be explicitly declared in ossec.conf
- Default Wazuh rules already provide strong baseline detection 
  without any custom rule-writing

## Files in This Repo
- screenshots/ — dashboard evidence (Overview, Endpoints, MITRE ATT&CK, detection events, custom rule)
- local_rules.xml — custom Wazuh detection rule
- wazuh-report.pdf — generated Wazuh threat hunting report

## Tools Used
VirtualBox, Ubuntu Server, Windows 7, Wazuh 4.9.2, Sysmon (SwiftOnSecurity config)
