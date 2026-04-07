# AI & Judicial State Capacity — Project Workflow

## Overview

This project studies the impact of AI transcription tools (specifically the Adalat AI platform) on judicial productivity and state capacity in Indian district courts. The research uses a randomized controlled trial (RCT) design across multiple Indian states — **Odisha, Bihar, Madhya Pradesh, Punjab, Haryana, and Chandigarh** — assigning judges to treatment (AI tool access) and control groups to measure outcomes such as case throughput, adjournment rates, and courtroom efficiency.

---

## Project Structure

```
ai_transcription/
├── code/
│   ├── odisha/                  # Odisha-specific cleaning and analysis code
│   │   ├── 00_preamble.do       # Global settings, packages, and directory macros
│   │   ├── admin data/          # Scripts to clean court admin data
│   │   ├── courtroom_observation/
│   │   ├── digital survey/
│   │   └── enumerator_survey/
│   └── randomization code/      # Treatment assignment scripts by state
│       ├── odisha/
│       ├── bihar/
│       ├── haryana/
│       ├── punjab/
│       ├── chandigarh/
│       ├── mp/
│       └── archive/
├── od_data/                     # Odisha data (raw, intermediate, clean, output)
│   ├── admin/
│   ├── courtroom_observation/
│   ├── daily_digital_survey/
│   ├── enumerator_survey/
│   ├── mixed_panel/             # App usage data from Adalat AI platform
│   └── output/                  # Final graphs and reports
└── rand_data/                   # Judicial officer rosters and treatment assignments
```

---

## Workflow

### Step 1 — Randomization (Treatment Assignment)

**Scripts:** `code/randomization code/<state>/randomize_<state>.do`

- Import judicial officer rosters from Excel files in `rand_data/` (e.g., `CJJD-judicial_officers1.xlsx`, `CJSD-judicial_officers2.xlsx`, state-specific lists)
- Stratify by judge designation (e.g., Junior Civil Judge, Senior Civil Judge) and/or district
- Draw random samples using `set seed 1234567` for reproducibility
- Assign 50/50 treatment vs. control within strata
- Export final assignment lists to `rand_data/treatment_assignment_<state>.xlsx` and `treated districts_<state>.xlsx`

**Pilot randomization** (`randomize_pilot.do`): Selected 80 Odisha judges (40 treatment / 40 control) drawn equally from two court lists (CJJD and CJSD), stratified 20T/20C per source.

---

### Step 2 — Data Collection (Three Sources)

#### 2a. Administrative Court Data (Cause Lists & Orders/Judgements)

Raw data scraped from the Odisha district court portal:

- **Cause lists** (`od_data/admin/raw/cause_list/csv/`): Daily CSV files (one per court day, named `YYYYMMDD.csv`) listing all scheduled cases per judge/court
- **Orders & Judgements** (`od_data/admin/raw/orders_judgements/`): CSV files with case outcome records, containing encoded CNO (case number) links

#### 2b. Daily Digital Survey

Judges self-report daily court activity (cases listed, disposed, witnesses, arguments, adjournments, time on manual writing) via a Google Form, separately for civil and criminal matters. Friday responses are collected via a separate form.

**Raw files:** `od_data/daily_digital_survey/raw/*.csv`

#### 2c. Enumerator Survey

Field enumerators visit courtrooms and interview judges, courtroom staff, lawyers, and litigants about technology use, workload, digital infrastructure, and attitudes toward AI. Three survey versions (V1, V2, V3) were deployed over the study period.

**Raw files:** `od_data/enumerator_survey/raw/`

#### 2d. Courtroom Observation

Direct observation data collected in courtrooms (case hearings, adjournment reasons, procedural stages).

**Raw file:** `od_data/courtroom_observation/raw/`

#### 2e. Mixed Panel (App Usage Data)

Weekly CSV exports from the Adalat AI platform dashboard tracking recording minutes per user, by district, language, and app module (Judge Chamber dictaphone vs. live courtroom recording).

**Raw files:** `od_data/mixed_panel/<date range>/`

---

### Step 3 — Data Cleaning

All cleaning is done in Stata (`.do` files). Run from the master file:

```
code/odisha/admin data/0_master.do
```

Which calls in sequence:

#### 3a. Decode CNO (`00_decode cno.py`)

Python script that decodes Base64-encoded URLs in the raw orders/judgements CSVs to extract the `cno` (case number), `order_no`, and `order_date` fields. Outputs `decoded_cno_YYYYMMDD.csv` to the intermediate folder.

#### 3b. Clean Orders & Judgements (`01_clean_orders_judgement.do`)

- Imports decoded CNO CSVs for each scrape date (currently `20250908`, `20251010`)
- Restricts to orders after 17 June 2025 (post-summer-vacation reopening)
- Standardizes `case_number`, `court_complex`, `court_name`
- Drops duplicates on `(cno, target_date)`
- Saves intermediate `.dta` files; appends and saves as `clean_orders_and_judgement.dta`

#### 3c. Clean Cause List (`02_clean_cause_list.do`)

- Loops over all daily CSV files in `od_data/admin/raw/cause_list/csv/`
- Handles format variations across scrape dates
- Standardizes string variables (removes line breaks, trims whitespace)
- Converts all date variables to Stata numeric format
- Extensively cleans and harmonizes categorical variables:
  - `purpose` (hearing purpose)
  - `stage_of_case` (procedural stage)
  - `case_type` (civil/criminal case classification)
- Appends all daily files into a single `clean_causelist.dta`

#### 3d. Merge Cause List with Orders/Judgements (`03_clean_causelist_orders_and_judgement_v1.do`)

- Merges the cleaned cause list with the cleaned orders/judgements dataset on `(cno, target_date)`
- Produces `od_data/admin/clean/merge_causelist_orders_and_judgement.dta`

#### 3e. Clean Enumerator Survey (`enumerator_survey/01_clean v1.do`, `02_clean_v2.do`, `03_clean_v3.do`)

- Drops non-judicial officer records and test submissions
- Labels variables for judge activities, hours spent, digital infrastructure, and AI attitudes
- Cleans and harmonizes responses across survey versions
- Produces respondent-level files by type: `respondent_judge.dta`, `respondent_courtroomstaff.dta`, `respondent_lawyer.dta`, `respondent_litigants.dta`
- Appended across versions into `clean_append_all.dta`

#### 3f. Clean Digital Survey (`digital survey/01_clean_digital_survey.do`)

- Imports Google Form CSV exports
- Renames raw columns to human-readable variable names
- Separates civil and criminal case statistics
- Cleans date variables and judge identifiers

#### 3g. Clean Courtroom Observation (`courtroom_observation/01_clean_courtroom_observation.do`)

- Imports courtroom observation `.dta`
- Cleans and labels observational variables

---

### Step 4 — Analysis & Output

**Output directory:** `od_data/output/`

- Graphs exported as `.png` to `od_data/output/graphs/`
- Summary reports exported as `.docx` to `od_data/output/`

Key outcome graphs include:
- Average cases per day and orders per day (by treatment status)
- Average judgements per week
- Adjournment reasons (visit-wise)
- Testimonies, hours spent, delays
- Digital infrastructure indicators (cloud, Wi-Fi, devices, IT staff)
- AI collaboration and meeting frequency measures

---

## Preamble / Directory Setup

The `00_preamble.do` detects the user's machine by checking known Dropbox paths and sets global macros:

| Macro | Points to |
|---|---|
| `$root` | User's Dropbox root |
| `$data` | `courts_transcription/odisha/data` |
| `$enumerator_survey` | `$data/enumerator_survey` |
| `$daily_digital_survey` | `$data/daily_digital_survey` |
| `$admin` | `$data/admin` |
| `$out` | `$data/output` |
| `$graphout` | `$out/graphs` |

**Configured for:** Meghana (`C:/Users/Meghana/Dropbox`), Nutan (`C:/Users/Nutan Satpathy/Dropbox`), Naila, Bharat, Kotia.

---

## Key Variables

| Variable | Description |
|---|---|
| `cno` | Unique court-level case number (CNO) |
| `target_date` | Cause list date / hearing date |
| `stage_of_case` | Procedural stage of the case |
| `purpose` | Purpose of the scheduled hearing |
| `case_type` | Civil/criminal case classification |
| `treatment` | 1 = assigned AI tool (Adalat AI), 0 = control |
| `judge_name` / `court_name` | Judge identifier |
| `district` / `court_complex` | Location identifiers |

---

## Tools & Languages

| Tool | Use |
|---|---|
| **Stata** | Data cleaning, merging, analysis, graph generation |
| **Python** | Decoding Base64-encoded CNO URLs from scraped admin data |
| **SurveyCTO** | Enumerator survey instrument and import |
| **Google Forms** | Daily digital survey collection |
| **Excel** | Judicial officer rosters, treatment assignment exports |
