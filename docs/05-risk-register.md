# 5. Risk register

Risks that are accepted rather than fully mitigated, with the reason and the plan. Re-reviewed whenever the design changes, and at least every 6 months.

| ID | Risk | Residual rating | Why accepted | Plan | Owner |
|---|---|---|---|---|---|
| I-11 | The server can decrypt user data (it holds the master key), so a full host compromise exposes everything | Medium | Server-side encryption already defeats the most common case, a leaked DB or backup. End-to-end encryption needs client-side crypto and key recovery design. | v2: encrypt in the browser with WebCrypto; the server stores only ciphertext. | Funso |
| R-02 | A DB-level attacker can delete audit rows | Low | Needs DB write access, which already implies a serious breach. | Ship audit events to append-only external storage (e.g. a WORM bucket). | Funso |
| I-04 | Registration reveals whether an email is registered | Low | Telling users "this email is taken" is needed for usability; rate-limited. | Move to email-verified sign-up, with "check your inbox" for every address. | Funso |
| D-03 | Attackers can lock out a known user for 15 minutes | Low | Temporary, and better than allowing unlimited guessing. | Add progressive delays and CAPTCHA before lockout. | Funso |
| D-04 | Rate limits are per process; a large distributed flood isn't stopped by the app | Low | Single-instance deployment; volumetric DDoS belongs at the edge. | Redis-backed limits if scaled; edge rate limiting on the host. | Funso |

## Rating method

Likelihood (1 rare – 3 likely) × Impact (1 minor – 3 severe), before controls in [STRIDE](02-stride.md). Residual rating is after controls: **Low** (accept), **Medium** (accept with a dated plan), **High** (must fix before release; none at v1.0).
