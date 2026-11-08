# TryHackMe – Simple CTF

> **Platform:** TryHackMe  
> **Target used during practice:** `10.49.162.233` (lab-instance IP; may change after reset)  
> **Progress:** Enumeration and exploitation steps documented from the practice session. Full completion and flag retrieval have not yet been confirmed.

This is a learning record for the Simple CTF room. Keep this write-up separate from the Boiler CTF walkthrough because they are different machines with different attack paths.

> **Lab safety:** Use these notes only against your assigned TryHackMe target or another system where you have explicit permission.

## 1. Service enumeration

The Nmap scan identified these services:

| Port | Service | Observation |
|---:|---|---|
| 21 | FTP | Anonymous login was available |
| 80 | HTTP | Web application |
| 2222 | SSH | SSH on a non-standard port |

Example enumeration command:

```bash
nmap -sC -sV -p- TARGET_IP
```

Replace `TARGET_IP` with the IP currently assigned by TryHackMe.

## 2. FTP enumeration

Anonymous FTP access exposed:

```text
/pub/ForMitch.txt
```

This file was part of the information-gathering stage.

## 3. Web application and SQL injection

The web application was identified as **CMS Made Simple**.

The investigation involved **CVE-2019-9053**, a SQL injection vulnerability affecting certain CMS Made Simple versions/configurations. During the lab, this path yielded credentials associated with the `mitch` account.

Keep any recovered lab credentials out of public commits. Do not publish passwords, private keys, session cookies, or flag values.

## 4. SSH access

SSH was available on port `2222`. The recovered `mitch` credentials were used during the lab to investigate SSH access.

Example connection syntax:

```bash
ssh -p 2222 mitch@TARGET_IP
```

Use only credentials recovered from your own authorized lab instance.

## 5. Privilege-escalation investigation

The next recorded investigation involved checking sudo permissions and a possible Vim-based privilege-escalation path.

Useful enumeration command:

```bash
sudo -l
```

**Completion note:** Earlier notes mention a sudo Vim path, but the available session record does not conclusively confirm the final privilege-escalation result or whether both flags were retrieved. Update this section only after verifying the exact commands and results from your own room session.

## 6. Findings recorded so far

- Anonymous FTP access exposed a file under `/pub`.
- The web service hosted CMS Made Simple.
- The investigation used the SQL injection issue tracked as CVE-2019-9053.
- Credentials for `mitch` were recovered in the lab.
- SSH was available on TCP port `2222`.
- A sudo/Vim privilege-escalation route was investigated.

## 7. Lessons learned

- Enumerate all TCP ports, not just common ports.
- Check anonymous FTP access and review exposed files.
- Identify the web application and verify its version before assessing known vulnerabilities.
- Treat recovered credentials as sensitive; do not publish them.
- Check sudo permissions and understand the security implications of privileged editors.
- Record evidence for each step and distinguish confirmed results from assumptions.

## 8. Further documentation to add

Before marking this room as completed, add your verified notes for:

- Exact Nmap command and relevant output
- CMS Made Simple version and affected component
- Exact, lab-safe reproduction steps and evidence for CVE-2019-9053
- SSH login confirmation
- `sudo -l` output and the verified Vim escalation result
- User/root flag file locations (do not publish flag values)
- Remediation recommendations

## Disclaimer

This is an educational write-up of an authorized TryHackMe practice environment. The target IP is specific to a lab instance and may change. Never test systems without explicit authorization.
