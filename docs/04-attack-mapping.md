# 4. MITRE ATT&CK and Cyber Kill Chain mapping

## MITRE ATT&CK (Enterprise)

How real-world adversary techniques would apply to Secure Vault, and what stops or detects each one.

| Tactic | Technique | How it would target the vault | Prevent | Detect | Threat |
|---|---|---|---|---|---|
| Reconnaissance | [T1595.002](https://attack.mitre.org/techniques/T1595/002/) Vulnerability Scanning | Scanners probe for known flaws | Patched deps and image (CI gate) | Request logs | T-06 |
| Reconnaissance | [T1589.002](https://attack.mitre.org/techniques/T1589/002/) Gather Victim Identity Info: Email Addresses | Check which emails have accounts | Uniform login errors and timing | `login_failed` audit events | I-03, I-04 |
| Initial Access | [T1190](https://attack.mitre.org/techniques/T1190/) Exploit Public-Facing Application | SQLi, XSS, IDOR against the app | ORM, escaping + CSP, owner-scoped queries | SAST/DAST in CI | E-03, I-05, T-02 |
| Initial Access | [T1078](https://attack.mitre.org/techniques/T1078/) Valid Accounts | Log in with stolen credentials | MFA | Login audit log with IPs | S-01 |
| Initial Access | [T1195.001](https://attack.mitre.org/techniques/T1195/001/) / [.002](https://attack.mitre.org/techniques/T1195/002/) Supply Chain Compromise | Poisoned package or GitHub Action | Version pins, SHA pins, checksums | Trivy, Dependabot, zizmor | T-05, T-06 |
| Credential Access | [T1110.001](https://attack.mitre.org/techniques/T1110/001/) Password Guessing | Online guessing against one account | Lockout after 5 failures | `account_locked` event | S-01 |
| Credential Access | [T1110.003](https://attack.mitre.org/techniques/T1110/003/) Password Spraying | One password, many accounts | Per-IP rate limit, common-password blocklist | `login_rate_limited` event | S-02 |
| Credential Access | [T1110.004](https://attack.mitre.org/techniques/T1110/004/) Credential Stuffing | Breach lists from other sites | MFA, rate limit | Spike in `login_failed` | S-01 |
| Credential Access | [T1110.002](https://attack.mitre.org/techniques/T1110/002/) Password Cracking | Offline cracking of a stolen DB | Argon2id (memory-hard) | - | I-01 |
| Credential Access | [T1539](https://attack.mitre.org/techniques/T1539/) Steal Web Session Cookie | XSS or malware grabs the session | `HttpOnly`, CSP, short lifetimes | Unknown IP in audit log | S-05 |
| Credential Access | [T1111](https://attack.mitre.org/techniques/T1111/) Multi-Factor Authentication Interception | Phish and relay a TOTP code | Single-use codes (replay protection) | `mfa_failed` events | S-04 |
| Credential Access | [T1552.001](https://attack.mitre.org/techniques/T1552/001/) Unsecured Credentials: Credentials In Files | Master key committed to git | Env-only secrets, `.gitignore` | Gitleaks (full history) | I-02 |
| Defense Evasion / Persistence | [T1556.006](https://attack.mitre.org/techniques/T1556/006/) Modify Authentication Process: MFA | Turn off MFA after a session theft | Needs password + fresh code | `mfa_disabled` event, visible to user | E-02 |
| Defense Evasion | [T1070](https://attack.mitre.org/techniques/T1070/) Indicator Removal | Delete audit rows | App never deletes audit rows | ⚠️ Gap: needs external log shipping | R-02 |
| Collection | [T1213](https://attack.mitre.org/techniques/T1213/) Data from Information Repositories | Bulk-read notes after access | Per-user keys, owner scoping | `note_*`/`file_downloaded` events | T-02 |
| Exfiltration | [T1567](https://attack.mitre.org/techniques/T1567/) Exfiltration Over Web Service | Create share links to pull notes out | Links expire, view caps, revocable | `share_created` event, visible to user | I-07 |
| Impact | [T1499.003](https://attack.mitre.org/techniques/T1499/003/) Application Exhaustion Flood | Flood login (Argon2) or uploads | Rate limits before hashing, size caps | Error and latency metrics | D-01, D-02 |

## Cyber Kill Chain walkthrough

A realistic scenario: **credential stuffing against a vault user**, followed step by step to show where the chain breaks.

| Stage | Attacker action | Where Secure Vault breaks the chain |
|---|---|---|
| 1. Reconnaissance | Buys a combo list from an unrelated breach; checks which emails are registered | Login gives no hint whether an email exists (I-03). Registration does reveal it, but only 10 checks per IP per hour (I-04). |
| 2. Weaponization | Loads the list into a stuffing tool with rotating proxies | - (happens off-platform) |
| 3. Delivery | Sends sign-in attempts | 10 attempts per minute per IP; 5 failures lock the account for 15 min (S-01). |
| 4. Exploitation | One reused password works | **MFA stops it here** if enabled: a password-only session can't open any page (E-01). |
| 5. Installation | Tries to keep access: turns off MFA, or plants a long-lived session | Turning off MFA needs password + code (E-02). Sessions expire after 30 min idle or 12 h (S-05). |
| 6. Command & Control | Reuses the stolen session from attacker infrastructure | The new IP shows in the user's activity log; "Sign out everywhere" kills every session. |
| 7. Actions on Objectives | Shares or downloads notes | Every share and download is logged and visible. Links expire and can be revoked. Data is per-user encrypted, so no bulk access to other users. |

**Takeaway:** without MFA the chain breaks only at detection (stage 6). With MFA it breaks at exploitation (stage 4). That's why the app nudges users to turn MFA on from their account page, and why "make MFA mandatory" is on the roadmap.
