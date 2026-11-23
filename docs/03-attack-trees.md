# 3. Attack trees

Attack trees break an attacker's goal into the ways they could reach it. A path is only open if every step along it succeeds. A crossed-out step (✗) is blocked by a control, with the threat ID from [STRIDE](02-stride.md) in brackets.

## Goal 1: Read another user's notes

```mermaid
flowchart TD
    G["Read another user's notes"]

    G --> A["Take over their account"]
    G --> B["Bypass access control"]
    G --> C["Steal the data at rest"]
    G --> D["Abuse a share link"]

    A --> A1["Guess or stuff password ✗<br/>[S-01, S-02]<br/>rate limit + lockout"]
    A --> A2["Password phished<br/>(possible)"]
    A2 --> A2a["…but MFA code needed ✗<br/>[S-03, S-04]"]
    A --> A3["Steal session cookie"]
    A3 --> A3a["via XSS ✗<br/>[I-05] HttpOnly + CSP"]
    A3 --> A3b["via network ✗<br/>HTTPS + HSTS + Secure flag"]

    B --> B1["Change ID in URL (IDOR) ✗<br/>[T-02] owner-scoped queries"]
    B --> B2["SQL injection ✗<br/>[E-03] ORM + SAST gate"]
    B --> B3["CSRF to share their note to me ✗<br/>[T-01] CSRF token"]

    C --> C1["Steal DB backup"]
    C1 --> C1a["…but ciphertext only ✗<br/>[I-01]"]
    C1 --> C1b["…and steal master key too"]
    C1b --> C1c["From git ✗ [I-02] Gitleaks"]
    C1b --> C1d["From the image ✗ [I-02] env-only"]
    C1b --> C1e["From host secret store<br/>⚠️ residual: needs a host compromise"]

    D --> D1["Guess a token ✗<br/>[S-07] 256-bit + rate limit"]
    D --> D2["Token leaks via Referer ✗<br/>[I-06]"]
    D --> D3["Victim forwards the link<br/>⚠️ residual: limited by expiry + views + revoke"]

    classDef blocked fill:#14532d,stroke:#16a34a,color:#fff
    classDef residual fill:#78350f,stroke:#d97706,color:#fff
    class A1,A2a,A3a,A3b,B1,B2,B3,C1a,C1c,C1d,D1,D2 blocked
    class C1e,D3 residual
```

**Residual paths:** stealing both the database *and* the host's secret store (needs a host compromise, see R-02 and I-11), and social engineering a recipient into forwarding a live link (limited by expiry, view caps and revocation).

## Goal 2: Get malicious code into production

```mermaid
flowchart TD
    G["Ship a backdoor to production"]
    G --> P["Through the code"]
    G --> S["Through the supply chain"]

    P --> P1["Open a PR with obvious vulnerable code ✗<br/>[T-04] Semgrep/Bandit gate"]
    P --> P2["Commit a secret to use later ✗<br/>Gitleaks, full history"]
    P --> P3["Push straight to main ✗<br/>branch ruleset requires the gate"]
    P --> P4["Subtle logic bug that passes scanners<br/>⚠️ residual: code review + tests"]

    S --> S1["Compromised PyPI package"]
    S1 --> S1a["Known CVE ✗ [T-06] Trivy SCA"]
    S1 --> S1b["New malicious release ✗<br/>exact version pins"]
    S --> S2["Hijacked GitHub Action tag ✗<br/>[T-05] SHA pinning"]
    S --> S3["Tampered scanner binary ✗<br/>[T-05] SHA-256 checksum check"]
    S --> S4["Vulnerable base image ✗<br/>Trivy image scan + OS patching"]

    classDef blocked fill:#14532d,stroke:#16a34a,color:#fff
    classDef residual fill:#78350f,stroke:#d97706,color:#fff
    class P1,P2,P3,S1a,S1b,S2,S3,S4 blocked
    class P4 residual
```
