# Incident Report 001

## Detection and Investigation of RDP Brute-Force Attack Against Active Directory User

---

## Executive Summary

On **Feb 13 2026**, multiple failed Remote Desktop Protocol (RDP) login attempts were detected against the domain user **johnson** from source IP **192.168.0.194**.

Splunk correlation identified **more than 5 failed login attempts (Event ID 4625, Logon Type 3)** within a short timeframe, triggering a brute-force alert.

Investigation confirmed malicious activity originating from a Kali Linux attacker machine within the lab environment.

The incident was classified as a **True Positive brute-force attack** mapped to **MITRE ATT&CK T1110 – Brute Force**.

---

## Lab Environment

| Role              | Machine             | IP Address   |
| ----------------- | ------------------- | ------------ |
| Domain Controller | Windows Server 2022 | 192.168.0.104 |
| Domain Client     | Windows 10          | 192.168.0.105 |
| Attacker          | Kali Linux          | 192.168.0.194 |
| SIEM              | Splunk Enterprise   | 192.168.0.135 |

---

## Detection Logic

### Trigger Condition

> 5+ failed authentication attempts (Event ID 4625, Logon Type 3 or 10) from a single source IP within a short timeframe.

### Splunk Query

```spl
index=wineventlog EventCode=4625
| search Logon_Type=3 OR Logon_Type=10
| stats count by Account_Name, Source_Network_Address, Logon_Type
| where count > 5
```

### Why This Threshold?

* Normal users rarely fail RDP login more than 2–3 times.
* 5+ failures suggests password guessing or brute-force behavior.
* Logon Type 10 ensures detection of full RDP session attempts.
* Including Logon Type 3 ensures detection of pre-authentication brute-force attempts that do not result in full RDP session creation.
* This threshold balances detection sensitivity while minimizing false positives.

---

## Attack Execution (Red Team Simulation)

The attacker used Hydra from Kali Linux to perform a brute-force attack against the Windows 10 domain client.

```bash
hydra -t 4 -V -f -l johnson -P passwords.txt rdp://192.168.0.105
```

![Hydra Bruteforce](../assets/hydra_bruteforce.png)
Hydra terminal output showing repeated failed login attempts.

---

## Evidence Collection

### Windows Security Event Logs

* Event ID: 4625 (Failed Logon)
* Logon Type: 3 (Network Logon – authentication attempt via NLA)
    -> The failed logons were recorded as Logon Type 3, indicating authentication attempts occurred at the network layer (NLA) before full RDP session establishment.
* Target Account: johnson
* Source Network Address: 192.168.0.194

![Event Viewer 4625](../assets/eventviewer_4625.png)
Event Viewer showing multiple failed logon events.

---

### Splunk Correlation Results

The query returned the following:

| Account_Name | Source_Network_Address | Count |
| ------------ | ---------------------- | ----- |
| johnson     | 192.168.0.194           | 75    |

![Splunk Detection](../assets/splunk_detection.png)
Splunk search results showing failed login count exceeding threshold.

---

## Triage & Analysis

Findings:

* Multiple failed login attempts in rapid succession
* All attempts originated from a single external IP
* No indication of legitimate user activity during timeframe
* Behavior consistent with automated brute-force attack

This activity deviates from normal authentication patterns and indicates malicious intent.

---

## Incident Classification

* **Status:** True Positive
* **Technique:** Brute Force
* **MITRE ATT&CK Mapping:** T1110 [T1110.001 Password Guessing]

No containment actions were executed as this was a controlled lab simulation.

---

## Recommended Containment & Response Actions

Based on the investigation findings, the following containment actions are recommended in a production environment:

- Temporarily lock or disable the targeted user account.
- Reset the user password and enforce strong password policy.
- Block the malicious source IP at the firewall.
- Review and enforce account lockout policies.
- Implement Multi-Factor Authentication (MFA) for RDP access.

---

## Lessons Learned

* RDP should not be externally exposed without MFA.
* Strong account lockout policies reduce brute-force risk.
* Detection thresholds must be tuned to environment baseline.
* SIEM correlation enables rapid detection and response.

---

## Skills Demonstrated

* Active Directory Security Monitoring
* Splunk Log Analysis
* Windows Event Log Investigation
* MITRE ATT&CK Mapping
* Incident Triage & Response
* Detection Engineering Fundamentals
