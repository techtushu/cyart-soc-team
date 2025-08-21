# Phishing Response Checklist

This checklist guides SOC analysts when handling phishing incidents.

---

## 1. Detection
- [ ] Review SIEM alerts (Wazuh, Suricata, etc.)
- [ ] Validate log source and authenticity
- [ ] Check if custom phishing rules triggered

## 2. Analysis
- [ ] Identify targeted user(s)
- [ ] Collect evidence (email headers, logs, URLs, attachments)
- [ ] Verify if user clicked or submitted credentials

## 3. Containment
- [ ] Block malicious domain/IP at firewall or proxy
- [ ] Quarantine suspicious email in mail server
- [ ] Disable affected user account (if compromise suspected)

## 4. Eradication
- [ ] Remove phishing artifacts (emails, payloads)
- [ ] Apply threat intelligence lookups (VirusTotal, MISP)

## 5. Recovery
- [ ] Reset affected user credentials
- [ ] Restore services/accounts if disabled
- [ ] Monitor for post-phishing lateral movement

## 6. Lessons Learned
- [ ] Conduct awareness training for user(s)
- [ ] Update detection rules & playbooks
- [ ] Document response in incident tracker
