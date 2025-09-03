# 🚨 Incident Escalation Workflows

---

## 🏷 Escalation Tiers
| Tier | Role | Responsibility |
|------|------|----------------|
| **Tier 1** | Analyst | Alert triage, basic investigation |
| **Tier 2** | Investigator | Deep dive, correlation, log review |
| **Tier 3** | Specialist/Threat Hunter | Advanced forensics, malware analysis |

---

## 📊 Escalation Criteria
- ⚡ **Severity** → Critical vs Low.  
- 🧩 **Complexity** → Requires special skills/tools.  
- 🏢 **Impact** → Business-critical systems affected.  

---

## 📡 Communication Protocol
```mermaid
sequenceDiagram
    participant Tier1
    participant Tier2
    participant Tier3
    participant Manager
    Tier1->>Tier2: Escalate suspicious alert
    Tier2->>Tier3: Request advanced analysis
    Tier3->>Manager: Report confirmed incident


## Key Objectives
- Escalate incidents with minimal delay.  
- Standardize communication across SOC tiers.  
- Integrate automation to reduce workload.  

## References
- NIST SP 800-61 Incident Handling Guide.  
- SANS Incident Handler’s Handbook.  
- Splunk SOAR documentation.  
