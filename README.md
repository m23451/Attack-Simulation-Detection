# Attack-Simulation-Detection
SOC Home Lab: SIEM Build, Attack Simulation & Detection
A self-contained Security Operations Centre (SOC) home lab built to
practise SIEM administration, Windows endpoint telemetry collection,
attack simulation, detection engineering and security investigation.
Project Overview
This project uses Splunk Enterprise as the central SIEM and a Windows 11
virtual machine as the monitored endpoint. Windows security events,
Sysmon telemetry and PowerShell activity are collected by the Splunk
Universal Forwarder and sent to Splunk for indexing, searching and
visualisation.

The lab provides a controlled environment for establishing a baseline of
endpoint activity, generating ATT&CK-aligned test behaviour and
developing searches to identify suspicious activity.
Objectives
- Build an end-to-end endpoint-to-SIEM logging pipeline.
- Improve Windows visibility through native auditing, Sysmon and
  PowerShell Script Block Logging.
- Establish a baseline of normal endpoint activity before attack
  simulation.
- Develop and test Splunk Search Processing Language (SPL) detections.
- Troubleshoot telemetry collection, forwarding and ingestion issues.
- Document investigation evidence and lessons learned.
Architecture
The lab runs on a single laptop using VMware Workstation.
  Component               Role                    Configuration
  Splunk Enterprise       Central SIEM, search    Runs on the host;
and dashboards          receives forwarded
events on TCP 9997

  Windows 11 Enterprise   Monitored endpoint      192.168.56.20
Evaluation VM                                   

  VMware VMnet2           Isolated telemetry      Host-only network:
network                 192.168.56.0/24; host
address: 192.168.56.1

  NAT adapter             Internet access when    Separate from the
required                host-only telemetry
path
Splunk Universal        Forwards endpoint logs  Sends data to
Forwarder                                       192.168.56.1:9997
Data flow
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
Technologies
- VMware Workstation Pro -- virtualisation and VM snapshots.
- Windows 11 Enterprise Evaluation -- monitored endpoint.
- Splunk Enterprise -- SIEM, indexing, searching and dashboards.
- Splunk Universal Forwarder -- endpoint log forwarding.
- Sysmon -- enhanced Windows process and activity telemetry.
- Atomic Red Team -- controlled generation of ATT&CK-mapped
  behaviour.
- MITRE ATT&CK -- framework for categorising adversary techniques.
- PowerShell -- endpoint configuration and test execution.
Implementation
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

Example commands used:
auditpol /set /subcategory:"Process Creation" /success:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Other Object Access Events" /success:enable
Process command-line logging and PowerShell Script Block Logging were
also enabled to improve investigative visibility.

Example PowerShell Script Block Logging configuration:
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force

Set-ItemProperty `
  -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" `
  -Name "EnableScriptBlockLogging" `
  -Value 1 `
  -Type DWord
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

  Index           Data
  wineventlog   Windows Security and System events
sysmon        Sysmon Operational events
powershell    PowerShell Operational events
