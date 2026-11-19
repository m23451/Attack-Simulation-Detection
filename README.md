# Attack-Simulation-Detection
# SOC Home Lab: SIEM Build, Attack Simulation & Detection
A self-contained Security Operations Centre (SOC) home lab built to
practise SIEM administration, Windows endpoint telemetry collection,
attack simulation, detection engineering and security investigation.
## Project Overview
This project uses Splunk Enterprise as the central SIEM and a Windows 11
virtual machine as the monitored endpoint. Windows security events,
Sysmon telemetry and PowerShell activity are collected by the Splunk
Universal Forwarder and sent to Splunk for indexing, searching and
visualisation.

The lab provides a controlled environment for establishing a baseline of
endpoint activity, generating ATT&CK-aligned test behaviour and
developing searches to identify suspicious activity.
### Objectives
- Build an end-to-end endpoint-to-SIEM logging pipeline.
- Improve Windows visibility through native auditing, Sysmon and
  PowerShell Script Block Logging.
- Establish a baseline of normal endpoint activity before attack
  simulation.
- Develop and test Splunk Search Processing Language (SPL) detections.
- Troubleshoot telemetry collection, forwarding and ingestion issues.
- Document investigation evidence and lessons learned.
## Architecture

The lab runs on a single laptop using VMware Workstation.

### Environment Components

| Component | Role | Configuration |
|---|---|---|
| Splunk Enterprise | Central SIEM, search and dashboards | Runs on the host; receives forwarded events on TCP `9997` |
| Windows 11 Enterprise Evaluation VM | Monitored endpoint | `192.168.56.20` |
| VMware VMnet2 | Isolated telemetry network | Host-only network: `192.168.56.0/24`; host address: `192.168.56.1` |
| NAT adapter | Internet access when required | Separate from the host-only telemetry path |
| Splunk Universal Forwarder | Forwards endpoint logs | Sends data to `192.168.56.1:9997` |

### Data Flow

The following diagram illustrates how endpoint telemetry travels from the Windows VM to Splunk Enterprise.

```text
Windows 11 VM
  ├── Windows Security / System events
  ├── Sysmon Operational events
  └── PowerShell Operational events
           |
           v
Splunk Universal Forwarder
           |
           | TCP 9997 over VMware VMnet2
           v
Splunk Enterprise on host
  ├── wineventlog index
  ├── sysmon index
  └── powershell index
           |
           v
SPL searches, dashboards and detection investigations
```
## Technologies
- VMware Workstation Pro -- virtualisation and VM snapshots.
- Windows 11 Enterprise Evaluation -- monitored endpoint.
- Splunk Enterprise -- SIEM, indexing, searching and dashboards.
- Splunk Universal Forwarder -- endpoint log forwarding.
- Sysmon -- enhanced Windows process and activity telemetry.
- Atomic Red Team -- controlled generation of ATT&CK-mapped
  behaviour.
- MITRE ATT&CK -- framework for categorising adversary techniques.
- PowerShell -- endpoint configuration and test execution.
## Implementation
1. Network and virtual machine
A host-only VMware network (VMnet2) was configured with DHCP disabled:
- Host: 192.168.56.1
- Windows VM: 192.168.56.20
- Subnet: 192.168.56.0/24
- Splunk receiving port: TCP 9997
The Windows VM also has a NAT adapter for internet access when required.
The host firewall allows inbound TCP traffic on port 9997 for Splunk
forwarding.
2. Windows security auditing
Windows auditing was enabled to collect events relevant to
authentication, process execution and account management.

3. Sysmon
Sysmon was installed on the Windows VM using a community configuration
based on SwiftOnSecurity.
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
The Sysmon service and the Microsoft-Windows-Sysmon/Operational event
channel were checked locally before testing forwarding. This helped
distinguish endpoint telemetry-generation problems from SIEM ingestion
problems.
4. Splunk and Universal Forwarder
Splunk Enterprise was configured to receive forwarded events on TCP
9997.

Three indexes were used to separate telemetry sources:

| Index | Data |
|---|---|
| `wineventlog` | Windows Security and System events |
| `sysmon` | Sysmon Operational events |
| `powershell` | PowerShell Operational events |

outputs.conf and inputs.conf after configuration changes, the Universal Forwarder was restarted and its forwarding status checked.

## Verification
End-to-end ingestion was verified in Splunk with:
index=* | stats count by index, sourcetype, host
Events from the Windows VM appeared in all three indexes, confirming
that telemetry was reaching Splunk through the forwarder.

Successful Windows logons were also checked using Event ID 4624.
## Baseline Dashboard
A seven-panel Splunk dashboard named **SOC Lab Baseline** was created to establish visibility before attack simulation.

| Panel | Purpose | SPL |
|---|---|---|
| Logon failures over time | Identify trends that may indicate brute-force activity | `index=wineventlog EventCode=4625 \| timechart span=1h count` |
| Logon failures by user | Identify accounts receiving repeated failed logons | `index=wineventlog EventCode=4625 \| stats count by user \| sort -count` |
| New process creations | Review process execution and command lines | `index=sysmon EventCode=1 \| table _time, host, User, Image, CommandLine \| sort -_time` |
| Top parent-child process pairs | Review process lineage for unusual relationships | `index=sysmon EventCode=1 \| stats count by ParentImage, Image \| sort -count \| head 20` |
| New local accounts | Identify account-creation activity | `index=wineventlog EventCode=4720 \| table _time, host, user` |
| PowerShell script block activity | Review PowerShell script content | `index=powershell EventCode=4104 \| table _time, host, ScriptBlockText \| sort -_time` |

## Event volume by index   Check ingestion volume  index=* \| stats count by index across indexes
Some panels depend on specific activity and may remain empty until the
relevant event occurs. For example, Event ID 4720 appears when a local
account is created. An empty panel should be checked against the source
event channel before being treated as an ingestion failure.
## Attack Simulation and Detection
Atomic Red Team was installed on the Windows 11 VM to generate
controlled behaviour mapped to MITRE ATT&CK.

The testing workflow is:
1. Select an ATT&CK technique and its Atomic Red Team test.
2. Run the test in the isolated lab.
3. Inspect the endpoint logs to identify the generated events.
4. Search for the events in Splunk.
5. Develop or refine an SPL detection.
6. Record the result, supporting evidence and potential false
   positives.
7. Revert the VM to the logging baseline snapshot between test
   batches.
