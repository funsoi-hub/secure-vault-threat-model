# 2. STRIDE analysis

Each threat has an ID, a rating, the control that addresses it, and where that control lives, so a reviewer can check every claim. Ratings are **L**ikelihood × **I**mpact on a 1-3 scale, scored *before* the control.

Links point to [secure-vault](https://github.com/funsoi-hub/secure-vault). Every control marked ✅ has an automated test.

## Spoofing (pretending to be someone else)

| ID | Threat | L×I | Control | Where | Status |
|---|---|---|---|---|---|
| S-01 | Credential stuffing with passwords leaked from other sites | 3×3 | TOTP MFA; per-IP rate limit (10/min); account lockout after 5 failures | `app/security.py` `RateLimiter`, `app/main.py` `login` | ✅ |
| S-02 | Password spraying (one common password across many accounts) | 3×3 | Per-IP rate limit; blocklist of common passwords at sign-up; 12-character minimum | `password_problems` | ✅ |
| S-03 | Brute-forcing the 6-digit TOTP code | 2×3 | 5 attempts per 5 minutes per account. With 3 valid codes (±30 s skew), that's about a 1 in 67,000 chance per 5-minute window | `limiters["mfa"]` | ✅ |
| S-04 | Replaying a TOTP code seen over a shoulder or phished | 2×3 | Each accepted time-step is stored; that code and any earlier one are refused | `verify_totp` | ✅ |
| S-05 | Session hijacking with a stolen cookie | 2×3 | `HttpOnly` (no JS access), `Secure`, `SameSite=Strict`; idle 30 min, absolute 12 h; "sign out everywhere" | `set_cookie`, `load_session` | ✅ |
| S-06 | Session fixation (attacker plants a known session ID before login) | 1×3 | Pre-login session destroyed; new token issued at login and again after MFA | `login`, `mfa_verify` | ✅ |
| S-07 | Guessing a share-link token | 1×3 | 256-bit random tokens; per-IP rate limit on `/s/*` | `create_share` | ✅ |

## Tampering (changing data or code)

| ID | Threat | L×I | Control | Where | Status |
|---|---|---|---|---|---|
| T-01 | Cross-site request forgery (CSRF): a malicious site makes the user's browser delete notes | 2×2 | Signed double-submit CSRF token on every form; `Origin` header check; `SameSite=Strict` | `checked_form` | ✅ |
| T-02 | Editing another user's note by changing the ID in the URL (IDOR) | 3×3 | Every query filters on `owner_id`; random 128-bit IDs | `owned_note`, `owned_file` | ✅ |
| T-03 | Database attacker modifies or swaps ciphertext between rows | 1×3 | AES-GCM authentication tag; associated data binds ciphertext to owner, record and field | `app/crypto.py` | ✅ |
| T-04 | Malicious code merged through a pull request | 2×3 | Required security gate; branch ruleset on `main`; review | devsecops-pipeline | ✅ |
| T-05 | Compromised GitHub Action rewrites a version tag (as with trivy-action, March 2026) | 2×3 | All actions pinned to commit SHAs; scanners downloaded with pinned SHA-256 checksums; zizmor lints workflows | `security-gate.yml` | ✅ |
| T-06 | Malicious or vulnerable PyPI package | 2×3 | Exact version pins; Trivy SCA in CI; Dependabot; weekly scheduled scan | `requirements.txt` | ✅ |

## Repudiation (denying an action)

| ID | Threat | L×I | Control | Where | Status |
|---|---|---|---|---|---|
| R-01 | User or attacker denies signing in, sharing or downloading | 2×2 | Audit events with timestamp, IP and user; shown to the user | `audit()`, `/account` | ✅ |
| R-02 | Attacker with DB write access deletes audit rows to hide activity | 1×2 | **Partly mitigated.** The app never deletes audit rows, but a DB-level attacker can. Next step: ship logs to append-only external storage. | `AuditEvent` | ⚠️ Accepted |

## Information disclosure (data leaking)

| ID | Threat | L×I | Control | Where | Status |
|---|---|---|---|---|---|
| I-01 | Database backup or dump is stolen | 2×3 | Notes, files, names and TOTP seeds are AES-256-GCM encrypted. Passwords are Argon2id. Session and share tokens are stored as SHA-256 hashes only. | `app/crypto.py`, `app/models.py` | ✅ |
| I-02 | Master key leaks (committed, baked into the image, printed in logs) | 2×3 | Read from env only; app refuses to start in production without it; Gitleaks scans full history; `.env` git-ignored and Docker-ignored | `app/config.py` | ✅ |
| I-03 | Account enumeration through login responses or timing | 3×1 | Same message and status for wrong password, unknown user and locked account. Unknown users are checked against a dummy hash so timing matches. | `verify_password` | ✅ |
| I-04 | Account enumeration through the registration form | 3×1 | **Accepted** (an "already exists" message is needed for usability). Rate-limited to 10 sign-ups per IP per hour. | `register` | ⚠️ Accepted |
| I-05 | Stored XSS steals note contents | 2×3 | Jinja2 auto-escaping; CSP `script-src 'none'` blocks all script even if escaping fails | `CSP` | ✅ |
| I-06 | Share token leaks through the `Referer` header or search-engine indexing | 2×2 | `Referrer-Policy: no-referrer`; `X-Robots-Tag: noindex` on share pages | `security_middleware` | ✅ |
| I-07 | Chat-app link previews "open" a one-view share link before the recipient does | 3×1 | Viewing needs a deliberate button press (POST with CSRF token); GET never spends a view | `shared_landing` | ✅ |
| I-08 | Uploaded HTML/SVG runs script on the vault's origin | 2×3 | Downloads always sent as `application/octet-stream` with `Content-Disposition: attachment`, plus `nosniff` | `download_file` | ✅ |
| I-09 | Sensitive pages cached by the browser or a proxy | 2×2 | `Cache-Control: no-store` on every non-static response | `security_middleware` | ✅ |
| I-10 | Clickjacking tricks a user into clicking "Create link" | 1×2 | `X-Frame-Options: DENY`; CSP `frame-ancestors 'none'` | `security_middleware` | ✅ |
| I-11 | A compromised server reads decrypted notes | 1×3 | **Accepted for v1.** The server holds the master key. Fix: client-side (end-to-end) encryption, on the roadmap. | - | ⚠️ Accepted |

## Denial of service (making it unavailable)

| ID | Threat | L×I | Control | Where | Status |
|---|---|---|---|---|---|
| D-01 | Huge request bodies exhaust memory | 2×2 | `Content-Length` required on writes and capped at the upload limit + 64 KB; uploads capped at 5 MB; note size capped | `security_middleware` | ✅ |
| D-02 | Login flood burns CPU on Argon2 hashing | 2×2 | Per-IP rate limit is checked *before* hashing | `login` | ✅ |
| D-03 | Attacker deliberately locks a victim out by failing their logins | 2×1 | **Accepted trade-off.** Lockout is temporary (15 min) and rate limits slow it down. | `login` | ⚠️ Accepted |
| D-04 | Distributed flood from many IPs | 2×2 | Out of scope for the app; handled by the host's edge or CDN. In-memory limits are per process (documented). | - | ⚠️ Accepted |

## Elevation of privilege (gaining more access)

| ID | Threat | L×I | Control | Where | Status |
|---|---|---|---|---|---|
| E-01 | Reaching protected pages with a password-only (pre-MFA) session | 2×3 | Sessions carry `mfa_verified`; every protected route requires it | `current_user` | ✅ |
| E-02 | Turning off MFA with a stolen session | 2×3 | Turning MFA off needs the current password **and** a fresh TOTP code | `mfa_disable` | ✅ |
| E-03 | SQL injection to read other users' rows | 2×3 | SQLAlchemy ORM with bound parameters only; Semgrep and Bandit in CI | `app/main.py` | ✅ |
| E-04 | Container breakout or tampering after an app compromise | 1×3 | Non-root user; app code owned by root, so it's read-only to the app; slim base image; image scanned by Trivy | `Dockerfile` | ✅ |
| E-05 | Host-header attacks poison generated share URLs | 1×2 | `TrustedHostMiddleware`, with `ALLOWED_HOSTS` set to the real domain in production (see `.env.example`) | `create_app` | ✅ |

## Summary

| | Threats | Mitigated | Accepted with reason |
|---|---|---|---|
| Spoofing | 7 | 7 | 0 |
| Tampering | 6 | 6 | 0 |
| Repudiation | 2 | 1 | 1 |
| Information disclosure | 11 | 9 | 2 |
| Denial of service | 4 | 2 | 2 |
| Elevation of privilege | 5 | 5 | 0 |
| **Total** | **35** | **30** | **5** |

Accepted risks are tracked in the [risk register](05-risk-register.md).
