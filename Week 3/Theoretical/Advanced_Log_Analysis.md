# 📊 Advanced Log Analysis

---

## 🔑 Core Concepts
| Concept              | Description | Example |
|----------------------|-------------|---------|
| **Log Correlation**  | Linking events across sources | Failed logins (4625) + unusual outbound traffic |
| **Anomaly Detection** | Spot deviations from baseline | Login at 3 AM from unusual location |
| **Log Enrichment**   | Add context to raw logs | IP → Country mapping, user role, asset criticality |

---

## 🛠 Techniques
- 📌 **Time-based correlation** → Detect lateral movement by analyzing event timelines.  
- 📌 **Pattern matching** → Identify brute force attempts, repeated failed logins.  
- 📌 **Contextual enrichment** → Match IPs/domains with threat intel feeds.  

---

## 🎯 Objectives
- Strengthen cross-source visibility.  
- Detect complex threats missed by isolated events.  
- Minimize false positives using enrichment.  

---

## 📚 References
- [SANS Log Analysis Whitepaper](https://www.sans.org/white-papers/)  
- [Elastic Anomaly Detection](https://www.elastic.co/guide/en/machine-learning/)  
- [CISA Incident Reports](https://www.cisa.gov/)  
