# Post‑Incident Analysis Documentation (Phishing Mock Incident)

**Author:** Tushar Yadav
**Environment:** Google Sheets, Draw\.io (PlantUML optional)
**Scope:** Post‑incident analysis for a mock phishing attack: RCA (5 Whys), Fishbone diagram, SOC metrics (MTTD/MTTR), and Lessons Learned action plan. This document is written as **repository-ready documentation** so a new person can pick it up and run with it.

---

## Repository layout (what should be in the repo)

```
/post-incident-analysis
├─ README.md
├─ /docs
│  ├─ RCA_PostIncident.csv
│  ├─ IncidentLog_PostIncident.csv
│  ├─ LessonsLearned_PostIncident.csv
│  
├─ /screenshots
│  └─post-incident.drawio.png           
└─)
```

> All CSVs in `/docs` are importable into Google Sheets. `fishbone_postincident.puml` is the PlantUML source for the fishbone diagram.

---

## Quick start — For a new person (step‑by‑step)

Follow these numbered steps exactly. No prior context required.

### 1) Obtain the repository

* Option A (recommended, CLI):

  ```bash
  git clone git@github.com:<ORG_OR_USERNAME>/post-incident-analysis.git
  cd post-incident-analysis
  ```
* Option B (no git): Download the provided ZIP and extract it.

### 2) Read the README

Open `README.md` in the project root. It contains the full context and links to files in `/docs`.

### 3) Open the RCA

* File: `/docs/RCA_PostIncident.csv`.
* Action: Open in Google Sheets (File → Import → Upload → Select file → Insert new sheet(s)).
* Confirm the sheet tab is named **RCA** and shows columns `Question` and `Answer`.
* Replace mock answers with real findings if available.

### 4) Load the Incident Log and verify timestamps

* File: `/docs/IncidentLog_PostIncident.csv`.
* Import into Google Sheets as a new sheet named **Incidents**.
* Confirm headers exist and columns are ordered: `IncidentID, IncidentStart, DetectionTimestamp, ResponseStart, ContainmentEnd, DetectionHours, ResponseHours, Notes`.

**Set the formulas** (put these formulas in row 2 of the corresponding columns):

* DetectionHours (F2):

  ```excel
  =IF(AND(B2<>"",C2<>""),(C2-B2)*24,"")
  ```
* ResponseHours (G2):

  ```excel
  =IF(AND(D2<>"",E2<>""),(E2-D2)*24,"")
  ```

**Example mock row** (already included):

* `IncidentID`: PH-001
* `IncidentStart`: `2025-09-01 09:00`
* `DetectionTimestamp`: `2025-09-01 11:00` → DetectionHours should calculate `2`
* `ResponseStart`: `2025-09-01 11:15`
* `ContainmentEnd`: `2025-09-01 15:15` → ResponseHours should calculate `4`

### 5) Create Metrics tab (MTTD / MTTR)

* Add a new sheet named **Metrics**.
* Enter MTTD and MTTR cells and reference the Incidents sheet:

  * MTTD (cell B2):

    ```excel
    =AVERAGE(Incidents!F2:F100)
    ```
  * MTTR (cell B3):

    ```excel
    =AVERAGE(Incidents!G2:G100)
    ```
* If you have only one incident, these cells will show 2 and 4 respectively.
* Paste the 50-word summary (provided in `README.md`) into the Metrics sheet.

### 6) Review & export Fishbone Diagram

You can either render the PlantUML file or create the fishbone in Draw\.io.

**Option A — Render PlantUML (recommended for exact reproducibility):**

* Requirements: Java + plantuml.jar (download from PlantUML site).
* Command:

  ```bash
  # in repo root where fishbone_postincident.puml is located
  java -jar plantuml.jar docs/fishbone_postincident.puml
  ```
* Output: `docs/fishbone_postincident.png` or `fishbone_postincident.svg` will be generated. Move it to `/exports/fishbone.png`.

**Option B — Draw\.io manual recreation:**

* Open [https://app.diagrams.net](https://app.diagrams.net) → Create blank diagram → Build a right‑facing fishbone with branches: People, Process, Technology, Policy, Environment.
* Export → PNG → Save as `/exports/fishbone.png`.

### 7) Populate Lessons Learned

* File: `/docs/LessonsLearned_PostIncident.csv` → Import into Google Sheets as **LessonsLearned**.
* Columns: `Action Item, Owner, Priority, Due Date, Status`.
* Assign owners and realistic due dates. Mark `Status` as `Open` / `In Progress` / `Done`.

### 8) Finalize the report

* Ensure the following are complete:

  * RCA tab filled in.
  * Incidents tab timestamps correct and Detection/Response hours calculated.
  * Metrics tab shows MTTD / MTTR and contains the 50‑word summary.
  * LessonsLearned tab has owners and dates.
  * Fishbone PNG is in `/exports`.
* If required, export `Metrics` and `LessonsLearned` to PDF for inclusion in a formal report.

### 9) Commit changes to the repo (if you made edits locally)

```bash
git add docs/RCA_PostIncident.csv docs/IncidentLog_PostIncident.csv docs/LessonsLearned_PostIncident.csv exports/fishbone.png README.md
git commit -m "Update RCA, incidents, lessons learned and add fishbone export"
git push origin main
```

### 10) If you do not use CLI — use GitHub web UI

* Go to the repo on GitHub → Add file → Upload files → select the edited CSV/PNG files → Commit directly to `main` or open a Pull Request.

---

## Guidance for reviewers / acceptance criteria

Before marking the task as complete, confirm:

1. RCA is not empty and contains 5 Why questions with plausible answers.
2. Incident timestamps are valid datetimes and `DetectionHours` / `ResponseHours` compute correctly.
3. MTTD & MTTR are present in the Metrics tab and the 50-word summary is visible.
4. LessonsLearned contains assigned owners with due dates and at least one high-priority fix (e.g., email filtering).
5. Fishbone diagram PNG is attached to `/exports` and is export-quality.

---

## Troubleshooting & Notes

* **Date/time formatting:** If Detection or Response cells appear as text, change cell format to Date time (Format → Number → Date time) before applying formulas.
* **PlantUML rendering errors:** Ensure `plantuml.jar` version matches Java version. If errors persist use the PlantUML online editor and copy the output PNG into `/exports`.
* **Google Sheets sharing:** Use `Share → Anyone with link (Viewer)` so reviewers can view. If sensitive, restrict to the organisation domain.

