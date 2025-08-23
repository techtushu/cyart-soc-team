Evidence Preservation Task
📌 Overview

This repository contains the results of a digital forensics evidence preservation exercise. The goal of the task was to practice collecting volatile and non-volatile evidence using Velociraptor and ensuring integrity with proper chain-of-custody documentation.

🛠️ Tools Used

Velociraptor – For volatile data collection (netstat, memory acquisition)

FTK Imager – For evidence verification (optional step)

PowerShell – For generating SHA-256 hashes

🔍 Activities Performed

Volatile Data Collection

Collected active network connections using Velociraptor (SELECT * FROM netstat()).

Saved output as netstat_connections.csv.

Memory Acquisition

Collected a full memory dump from the VM using Artifact.Windows.Memory.Acquisition.

Saved as PhysicalMemory.dd (stored offline, not uploaded due to size).

Evidence Integrity

Generated SHA-256 hash values for both the memory dump and netstat CSV.

Documented them in the Chain of Custody table.

Documentation

Maintained chain-of-custody in chain_of_custody.csv.

Stored hash outputs in memory_dump_hash.txt.

📂 Repository Structure
Evidence-Preservation-Task/
├── All Windows.Network.Netstat.csv     # Volatile data collected
├── memory_dump_hash.txt        # SHA-256 hash of memory dump
├── chain_of_custody.csv        # Evidence log
├── README.md                   # This documentation
└── screenshots/                # Screenshots of Velociraptor/PowerShell

📑 Chain of Custody (Sample)
Item	Description	Collected By	Date	Hash Value
Memory Dump	Server-X full RAM dump (18 GB)	SOC Analyst	2025-08-23	<SHA256>
Netstat CSV	Volatile network connections (CSV)	SOC Analyst	2025-08-23	<SHA256>
⚠️ Notes

Full memory dump (PhysicalMemory.dd) and mem_acq.zip are not uploaded due to large size.

Only metadata, CSV, and hash files are preserved here.
