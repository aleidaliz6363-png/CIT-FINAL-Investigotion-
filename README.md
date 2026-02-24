# CIT-FINAL-Investigotion-
Splunk-Honeypot-Lab
**Course:** Digital Forensics and Incident Response  
**Program:** Cybersecurity Professional Bootcamp — University of Michigan  
**Author:** Aleidaliz Ramirez Perez  

---

## 📋 Overview

This project documents a full cybersecurity investigation completed as part of the CIT_FINAL capstone lab. The objective was to gain unauthorized-but-simulated access to a mail server, extract hidden credentials, analyze honeypot logs using Splunk, and capture a Base64-encoded flag by hunting through Pastebin URLs found in attacker download activity.

**Skills demonstrated:**
- Network port scanning and connectivity testing (PowerShell)
- Mail server access via Telnet and POP3 protocol
- SIEM log analysis using Splunk
- Cowrie honeypot log investigation
- Regex extraction and field parsing in Splunk
- PowerShell scripting for automated URL fetching
- Base64 decoding and flag capture

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| PowerShell | Port testing, scripting, Base64 decoding |
| Telnet / POP3 | Mail server access |
| Splunk | SIEM log analysis |
| Cowrie Honeypot | Attacker activity logs |
| Regex (Splunk `rex`) | Extracting Pastebin IDs from raw logs |
| Pastebin API | Retrieving attacker-uploaded payloads |

---

## 🔬 Investigation Steps

### Step 1: Test Server Connectivity

Checked which ports were open on the target server using PowerShell:

```powershell
Test-NetConnection 10.0.167.237 -Port 110
Test-NetConnection 10.0.167.237 -Port 8000
Test-NetConnection 10.0.167.237 -Port 25
```

**Result:** All three ports returned `TcpTestSucceeded: True`

---

### Step 2: Enable Telnet Client

```powershell
dism /online /Enable-Feature /FeatureName:TelnetClient
```

**Result:** Operation completed successfully.

---

### Step 3: Access Mail Server and Read Emails

Connected to the POP3 mail server via Telnet:

```
telnet 10.0.167.237 110
```

Authenticated and retrieved all emails:

```
USER johnd
PASS toor
LIST
RETR 1
RETR 2
RETR 3
RETR 4
QUIT
```

> ⚠️ **Key Finding:** Email #4 (subject: *"We launched Splunk!"*) contained Splunk login credentials.

---

### Step 4: Log into Splunk

Navigated to: `http://10.0.167.237:8000`

- **Username:** admin  
- **Password:** CTF_Final!

---

### Step 5: Set Time Range to All Time

In Splunk Search, clicked the time picker (top right) → Selected **All time** under OTHER.

> ⚠️ **Important:** Without this setting, historical events are hidden and results will be incomplete.

---

### Step 6: Search Cowrie Honeypot Logs

```splunk
index=* cowrie
```

**Result:** 56,972 events found

---

### Step 7: Search for Download Activity

```splunk
index=* cowrie download
```

**Result:** 28 download events identified

---

### Step 8: Search for wget/curl Commands

```splunk
index=* cowrie (wget OR curl)
```

**Result:** 28 command events — Pastebin URLs visible in the raw event data

---

### Step 9: Extract All Pastebin IDs with Regex

```splunk
index=* cowrie pastebin.com/raw/
| rex field=_raw "pastebin\.com/raw/(?<paste_id>[A-Za-z0-9]+)"
| stats count by paste_id
```

**Pastebin IDs discovered:**

| Paste ID | Occurrences | Notes |
|----------|-------------|-------|
| `0cs1NHvh` | 16 | ✅ **THIS ONE HAS THE FLAG** |
| `ZTWf2ZmP` | 16 | Decoy |
| `2QAuwmHe` | 8 | Decoy |
| `C4B5ZK8K` | 8 | Decoy |
| `jpSBiHjC` | 8 | Decoy |

---

### Step 10: Fetch Pastebin Content via PowerShell

```powershell
$ids = @("ZTWf2ZmP","C4B5ZK8K","0cs1NHvh","2QAuwmHe","jpSBiHjC")

foreach ($id in $ids) {
    $url = "https://pastebin.com/raw/$id"
    Write-Host "===== $id ====="
    try {
        $txt = (Invoke-WebRequest -UseBasicParsing $url).Content.Trim()
        Write-Host "Preview: " + $txt.Substring(0,[Math]::Min(120,$txt.Length))
    } catch {
        Write-Host "Failed to fetch $url"
    }
}
```

**Results:**

| Paste ID | Content |
|----------|---------|
| `0cs1NHvh` | `Q29uZ3JhdHMsIHlvdSBoYXZlIGZpbmlzaGVkIENJVF9GSU5BTCBzdWNjZXNzZnVsbHk=` |
| `ZTWf2ZmP` | "No, It's not it." |
| `2QAuwmHe` | "Downloading File ==========>> script.bat" |
| `C4B5ZK8K` | Decoy Base64 string |
| `jpSBiHjC` | Site rejected the request |

---

### Step 11: Decode the Base64 Flag

```powershell
[System.Text.Encoding]::UTF8.GetString(
  [System.Convert]::FromBase64String(
    "Q29uZ3JhdHMsIHlvdSBoYXZlIGZpbmlzaGVkIENJVF9GSU5BTCBzdWNjZXNzZnVsbHk="
  )
)
```

### 🏁 Flag Captured:

```
Congrats, you have finished CIT_FINAL successfully
```

---

## 📌 Key Takeaways

- POP3 mail servers can expose sensitive credentials if left unencrypted — always use secure protocols like IMAPS or SMTPS in production environments.
- Cowrie honeypots are highly effective at capturing attacker behavior including file downloads and command execution.
- Splunk's `rex` command is a powerful tool for extracting structured data from unstructured log fields.
- Attackers frequently use Pastebin and similar platforms to host payloads — monitoring for outbound connections to paste sites is a valuable detection strategy.
- Base64 encoding is not encryption — it is easily reversed and should never be used to protect sensitive data.

---

## 👩‍💻 Author

**Aleidaliz Ramirez Perez**  
Cybersecurity Professional Bootcamp — University of Michigan  
[LinkedIn](https://linkedin.com/in/aleidaliz-ramirez-perez) | [GitHub](https://github.com/aleidaliz)
