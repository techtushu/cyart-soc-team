1. Mean Time to Detect (MTTD)
Concept: This is the average time it takes to detect a threat from the moment it enters your environment.
Formula: MTTD= 
Number of Incidents
∑(Detection Timestamp−Event Timestamp)
​
 

Calculation with Mock Data:

Incident 1:

Event Timestamp: 2025-09-17T20:34:28.318Z

Detection Timestamp (from @timestamp): 2025-09-17T20:34:28.319Z

Time to Detect: 0.001 seconds

Incident 2:

Event Timestamp: 2025-09-17T20:34:30.500Z

Detection Timestamp (from @timestamp): 2025-09-17T20:34:30.505Z

Time to Detect: 0.005 seconds

Incident 3:

Event Timestamp: 2025-09-17T20:34:45.100Z

Detection Timestamp (from @timestamp): 2025-09-17T20:34:45.102Z

Time to Detect: 0.002 seconds

Total Detection Time: 0.001 + 0.005 + 0.002 = 0.008 seconds
Number of Incidents: 3

MTTD Calculation: 0.008 seconds/3=0.0027 seconds

2. Mean Time to Respond (MTTR)
Concept: This is the average time it takes for your security team to fully contain and resolve a detected incident.
Formula: MTTR= 
Number of Incidents
∑(Resolution Timestamp−Detection Timestamp)
​
 

Calculation with Mock Data:

Incident 1:

Detection Time: 20:34:28.319Z

Simulated Resolution Time: 21:34:28.319Z

Response Time: 1 hour

Incident 2:

Detection Time: 20:34:30.505Z

Simulated Resolution Time: 23:34:30.505Z

Response Time: 3 hours

Incident 3:

Detection Time: 20:34:45.102Z

Simulated Resolution Time: 00:34:45.102Z (next day)

Response Time: 4 hours

Total Response Time: 1 + 3 + 4 = 8 hours
Number of Incidents: 3

MTTR Calculation: 8 hours/3=2.67 hours

3. False Positive Rate
Concept: This is the percentage of alerts that are incorrectly flagged as malicious.
Formula: False Positive Rate= 
Number of Total Alerts
Number of False Positives
​
 ×100

Calculation with Mock Data:

Number of Total Alerts: 45 (based on the mock data)

Number of False Positives: 10 (alerts with _source.rule.level below 7)

False Positive Rate Calculation: (10/45)×100=22.22%

4. Dwell Time
Concept: This is the total time a threat actor is present in your network, from initial compromise to final detection.
Formula: Dwell Time=Discovery Timestamp−Initial Compromise Timestamp

Calculation with Mock Data:

Simulated Initial Compromise Timestamp: 2025-09-17T06:34:28.318Z

Discovery Timestamp: 2025-09-17T20:34:28.318Z

Dwell Time Calculation: 20:34 - 06:34 = 14 hours
