# 🌐 Threat Intelligence Integration

---

## 📂 Types of Threat Intelligence
| Type | Description | Example |
|------|-------------|---------|
| **IOCs** | Known malicious artifacts | IPs, domains, file hashes |
| **TTPs** | Attacker behavior patterns | MITRE ATT&CK techniques |
| **Feeds** | External TI sources | AlienVault OTX, MISP, STIX/TAXII |

---

## 🏗 SOC Integration Workflow
```mermaid
flowchart TD
A[Raw Alerts] --> B[Threat Feed Matching]
B --> C[Enrichment]
C --> D[Prioritized Alert]

## Key Objectives
- Improve detection and response with external intelligence.  
- Prioritize alerts based on context.  
- Transition from reactive → proactive threat hunting.  

## References
- MITRE ATT&CK Matrix.  
- OASIS STIX/TAXII Standards.  
- AlienVault OTX feeds.  
