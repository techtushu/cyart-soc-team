# Alert Priority Levels — Study Notes

## Priority Definitions
- **Critical**: Active or imminent impact causing major disruption or data loss (e.g., ransomware encryption, domain admin compromise).
- **High**: High-risk activity with strong indications of compromise (e.g., unauthorized admin access, malware beacon).
- **Medium**: Suspicious activity with limited scope (e.g., credential stuffing attempts).
- **Low**: Informational or low impact (e.g., port scans).

## Assignment Criteria
1. **Asset Criticality** — crown jewels > production > staging > test.
2. **Exploit Likelihood** — known CVE, public exploit, active exploitation.
3. **Business Impact** — financial, regulatory, reputation.
4. **Detection Confidence** — sensor fidelity and correlation.

## CVSS → SOC Priority
- 9.0–10.0 → Critical
- 7.0–8.9 → High
- 4.0–6.9 → Medium
- 0.1–3.9 → Low

## Examples
- Log4Shell (CVE-2021-44228) CVSS 9.8 → Critical.
- Unauthorized admin login → High/Critical depending on sensitivity.
- Ransomware encryption detected → Critical.
