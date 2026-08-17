# 🛡️ SOC Incident Report — Phishing Campaign & Account Compromise Investigation

---

**Report ID:** IR-2024-001
**Classification:** Confidential — Internal Use Only
**Analyst:** [Your Name]
**Date:** 2024
**Severity:** HIGH
**Status:** RESOLVED / CONTAINED

---

## 1. Executive Summary

A targeted phishing campaign was identified after a user reported a suspicious email impersonating a trusted organization (Cloudora). Investigation revealed a multi-stage attack involving a lookalike domain, credential harvesting, and successful account compromise. Two user accounts were confirmed compromised via impossible travel indicators correlated with credential submission events.

The phishing infrastructure was mapped, all affected sessions were revoked, credentials reset, MFA re-registered, and malicious domains/IPs blocked across security controls. A full email purge was performed across all recipient mailboxes. No further attacker activity was observed post-containment.

**Key Findings:**
- Phishing email passed SPF/DKIM/DMARC — authenticated against attacker-controlled lookalike domain
- 1 user clicked the phishing link and submitted credentials
- Impossible travel detected — successful foreign login 2.5 hours after credential submission
- Pivot on attacker IP revealed a second compromised account
- Full containment achieved within 35 minutes of compromise identification

---

## 2. Incident Timeline (UTC)

| Time (UTC) | Event | Evidence Source |
|---|---|---|
| 08:00 | Phishing email delivered to multiple recipients | Mail Gateway Logs |
| 08:05 | User A clicks phishing URL | Message Trace / URL Click Logs |
| 08:06 | User A submits credentials on phishing page | Phishing site interaction / form submit log |
| 08:07 | Analyst receives phishing report from User A | Helpdesk Ticket |
| 08:10 | Analyst begins email header analysis | Raw email source |
| 08:25 | Lookalike domain and attacker infrastructure identified | VirusTotal / AbuseIPDB |
| 08:30 | Message trace initiated — recipients and clickers mapped | Exchange / M365 Message Trace |
| 09:00 | Campaign scope confirmed — 3 recipients, 1 clicker, 1 credential submitter | Message Trace |
| 10:34 | Successful sign-in from Amsterdam (NL) on new device — User A normally in Manchester (UK) | Azure AD / Sign-in Logs |
| 10:35 | Impossible travel indicator flagged — account takeover suspected | Sign-in Logs |
| 10:36 | Pivot on attacker IP — second successful foreign login identified (User B) | Sign-in Logs |
| 10:40 | Sessions revoked — User A and User B | Identity Platform |
| 10:42 | Passwords reset — User A and User B | Identity Platform |
| 10:43 | MFA re-registered — User A and User B | Identity Platform |
| 10:45 | Phishing domain, subdomains, and attacker IPs blocked at email gateway, web proxy, DNS | Security Controls |
| 10:48 | Phishing emails purged from all recipient mailboxes | Mail Admin Console |
| 10:50 | Post-containment verification — no further attacker activity observed | Sign-in Logs / Gateway Logs |
| 11:00 | Incident report drafted and submitted | This document |

---

## 3. Investigation Methodology

### 3.1 Phase 1 — Email Header Analysis

The reported email was examined as **raw text** to avoid rendering tracking pixels or activating embedded content.

**Key headers examined:**

| Header | Value | Finding |
|---|---|---|
| From | security@cloudora-hortal.example | Lookalike domain — NOT cloudora.io |
| Return-Path | bounce@cloudora-hortal.example | Matches From — attacker-controlled |
| Reply-To | hr-support@attacker-domain.com | Mismatch — replies redirect to attacker |
| SPF | PASS | Passed for cloudora-hortal.example — attacker owns domain |
| DKIM | PASS | Signed by cloudora-hortal.example — not cloudora.io |
| DMARC | PASS | Aligned to cloudora-hortal.example |
| Received (bottom hop) | 198.X.X.X | Originating sending IP — documented as IOC |
| Subject | URGENT: Verify Your Credentials — Account Suspension |

> ⚠️ **Critical Finding:** All three authentication protocols (SPF/DKIM/DMARC) returned PASS. However, authentication was aligned to the **attacker-controlled domain** (cloudora-hortal.example), NOT the legitimate organization domain (cloudora.io). Authentication PASS does NOT equal legitimacy.

---

### 3.2 Phase 2 — Domain Analysis

**Legitimate domain:** `cloudora.io`
**Attacker domain:** `cloudora-hortal.example`

**Technique used:** Brand name embedded inside unrelated domain (typosquatting variant)

| Indicator | Detail |
|---|---|
| Domain creation date | Registered 3 days before phishing campaign |
| VirusTotal detections | 2/90 vendors flagged (newly registered — low detection expected) |
| AbuseIPDB confidence | 87% abuse confidence for sending IP 198.X.X.X |
| Hosting provider | Bulletproof hosting provider — known for abuse |
| Subdomains | login.cloudora-hortal.example (credential harvesting page) |

> ⚠️ **SOC Note:** Low VirusTotal detection does NOT confirm safety. Newly registered phishing domains routinely show 0–2 detections at time of campaign. Evidence-first analysis identified this as malicious before threat intelligence confirmed it.

---

### 3.3 Phase 3 — URL Analysis

**Displayed link text in email:** `Click here to verify your Cloudora account`

**Actual URL (defanged):** `hxxps://login.cloudora-hortal[.]example/verify?user=target@company.com`

**Analysis:**
- Subdomain designed to mimic legitimate login portal
- Pre-populated victim email address in URL parameter (personalized phishing)
- SSL certificate present — HTTPS does NOT indicate legitimacy
- Credential harvesting form confirmed on destination page (sandbox analysis)

---

### 3.4 Phase 4 — Social Engineering Indicators

| Technique | Evidence in Email |
|---|---|
| Urgency | "Your account will be suspended within 2 hours" |
| Authority | Impersonated security team of known vendor |
| Fear | "Failure to verify will result in data loss" |
| Credential request | Linked to fake login portal |
| Pressure deadline | "Immediate action required today" |

---

### 3.5 Phase 5 — Message Trace & Scope Mapping

**Recipients:** 3 total
**Delivered:** 3
**Quarantined:** 0 (email passed authentication — bypassed spam filter)
**Clicked:** 1 (User A)
**Credentials submitted:** 1 (User A)
**Compromised:** 2 (User A — direct; User B — identified via attacker IP pivot)

| Victim Level | User | Action Taken |
|---|---|---|
| Level 1 — Received only | User C | Monitored — no action required |
| Level 3 — Credentials submitted | User A | Full containment |
| Level 4 — Account compromised | User B | Full containment (pivot discovery) |

---

### 3.6 Phase 6 — Sign-in Log Analysis & Impossible Travel

**User A sign-in events post-phishing:**

| Time (UTC) | Location | IP | Device | Result |
|---|---|---|---|---|
| 07:55 | Manchester, UK | 82.X.X.X | Known Windows laptop | Success (legitimate) |
| 10:34 | Amsterdam, NL | 198.X.X.X | Unknown device | Success ⚠️ |

**Impossible travel calculation:**
- Manchester → Amsterdam = ~530 km
- Time elapsed = 2h 39min
- Physically impossible without air travel — no travel logged
- **Verdict: Account takeover confirmed**

**Pivot finding:**
Searching sign-in logs for all authentications from `198.X.X.X` revealed User B also authenticated successfully from the same attacker IP at 10:36 UTC — 2 minutes after User A's compromise.

---

## 4. Indicators of Compromise (IOCs)

| Type | Value | Context |
|---|---|---|
| Domain | cloudora-hortal.example | Phishing sender domain |
| Subdomain | login.cloudora-hortal.example | Credential harvesting page |
| IP Address | 198.X.X.X | Phishing email sending IP / Attacker login IP |
| URL (defanged) | hxxps://login.cloudora-hortal[.]example/verify | Phishing URL |
| Email address | security@cloudora-hortal.example | Phishing sender |
| Reply-To | hr-support@attacker-domain.com | Attacker reply address |

---

## 5. MITRE ATT&CK Mapping

| Technique ID | Technique Name | Observed Behavior |
|---|---|---|
| T1566.001 | Phishing: Spearphishing Attachment | Targeted phishing email delivered to specific users |
| T1566.002 | Phishing: Spearphishing Link | Malicious URL embedded in email body |
| T1598.003 | Phishing for Information: Spearphishing Link | Credential harvesting via fake login portal |
| T1078 | Valid Accounts | Attacker used stolen credentials for login |
| T1539 | Steal Web Session Cookie | Session hijacking risk post-credential theft |
| T1114 | Email Collection | Potential inbox access post-compromise |
| T1110 | Brute Force (credential stuffing risk) | Stolen credentials may be reused elsewhere |

---

## 6. Containment Actions Taken

Actions were performed in the following order to ensure complete eradication:

```
Step 1 → Revoke all active sessions (User A + User B)
            Reason: Active attacker sessions must be killed BEFORE password reset
            to prevent continued access via existing session tokens.
            ↓
Step 2 → Reset passwords (User A + User B)
            ↓
Step 3 → Re-register MFA (User A + User B)
            ↓
Step 4 → Block IOCs at security controls
            - Email gateway: sender domain + IPs blocked
            - Web proxy: phishing URL blocked
            - DNS: lookalike domain blocked
            - EDR: attacker IP added to blocklist
            ↓
Step 5 → Purge phishing emails from all recipient mailboxes (3 mailboxes)
            ↓
Step 6 → Post-containment verification
            - Checked sign-in logs: no further attacker activity
            - Confirmed blocked IOCs not generating new hits
            - Verified MFA re-registration completed by both users
```

---

## 7. Post-Containment Verification

| Check | Result |
|---|---|
| Further logins from attacker IP 198.X.X.X | None detected |
| New successful sign-ins from foreign locations | None detected |
| Phishing emails remaining in mailboxes | 0 — fully purged |
| New phishing variants from same infrastructure | None detected |
| User A account activity post-reset | Normal — legitimate activity only |
| User B account activity post-reset | Normal — legitimate activity only |

**Verdict: Fully contained.**

---

## 8. Recommendations

1. **Email filtering rule** — Create a rule to flag all emails from newly registered domains (< 30 days old) for additional scrutiny regardless of authentication results.

2. **User awareness training** — Conduct targeted phishing awareness training focused on the key lesson: SPF/DKIM/DMARC PASS does not mean the sender is trustworthy — the domain must be verified.

3. **Impossible travel alerting** — Implement automated alerting for impossible travel events in sign-in logs to reduce response time from detection to containment.

4. **MFA enforcement** — Enforce phishing-resistant MFA (FIDO2/hardware keys) rather than SMS or app-based OTP, which can be bypassed via real-time phishing proxies.

5. **Spam filter tuning** — Review why the phishing email bypassed quarantine despite being from a newly registered lookalike domain. Consider adding domain age as a spam scoring factor.

6. **IOC sharing** — Share confirmed IOCs (domain, IPs, URLs) with threat intelligence platforms and ISACs relevant to the industry.

---

## 9. Lessons Learned

| Lesson | Detail |
|---|---|
| Authentication ≠ Legitimacy | All three email authentication protocols passed — the email was still phishing. Domain alignment matters more than PASS/FAIL. |
| Low threat intel score ≠ Safe | VirusTotal showed only 2 detections at time of investigation. Evidence-first analysis identified threat before reputation confirmed it. |
| Always pivot on attacker IP | Investigating only User A would have missed User B's compromise entirely. |
| Session revocation before password reset | Resetting the password alone does not terminate active attacker sessions. |
| Containment order matters | The sequence of revoke → reset → MFA → block → purge → verify is critical for complete eradication. |

---

## 10. Analyst Notes

> This investigation was conducted as part of a simulated SOC analyst training exercise based on the TCM Security SOC 101 methodology and phishing investigation framework. All IP addresses, domain names, and user identifiers are fictional and used for educational demonstration purposes only.
>
> This report demonstrates proficiency in: email header analysis, SPF/DKIM/DMARC interpretation, IOC extraction and enrichment, message trace analysis, sign-in log correlation, impossible travel detection, attacker infrastructure pivoting, containment sequencing, and SOC report writing.

---

*Report prepared by: Shrutika Parsai | Cybersecurity Student | SOC Analyst in Training*
*Contact:  
