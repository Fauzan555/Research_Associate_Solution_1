# 🇮🇳 Indian Startup Incubators: Dual-Methodology Identification & Longitudinal Web Archival Pipeline (2022–2026)

![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue.svg)
![Data Engineering](https://img.shields.io/badge/Focus-Web%20Archival%20%26%20Panel%20Data-success.svg)
![Wayback Machine CDX](https://img.shields.io/badge/API-Wayback%20Machine%20CDX-orange.svg)
![Status](https://img.shields.io/badge/Deliverables-Complete-brightgreen.svg)

## 📌 Executive Summary
This repository contains the research design, empirical dataset, and automated Python web-archival pipeline developed to:
1. **Identify and Rank the Top 20 Startup Incubator Programs in India** using two independent, reproducible methodologies (**Top-Down Institutional Scale** vs. **Bottom-Up Portfolio Market Impact**).
2. **Automate Longitudinal Web Collection (Jan 2022 – Sep 2026)** of each incubator's **(i) Homepage** and **(ii) About Us page** across both **Current Live Websites** and **57 Monthly Historical Snapshots** via the Internet Archive **Wayback Machine CDX Server API**.

---

## 📂 Repository Structure
```text
├── README.md                                  # Detailed methodology, setup, and results documentation
├── top_20_indian_startup_incubators.xlsx      # Deliverable 1(a)(iii): Formatted spreadsheet of Top 20 incubators
├── top_20_indian_startup_incubators.csv       # Machine-readable input dataset for the Python scraper
├── incubator_wayback_scraper.py               # Deliverable 2(a)(i): Annotated Python 3 longitudinal scraper
├── requirements.txt                           # Python dependencies
└── scraped_html_archive/                      # Deliverable 2(a)(iii): Scraped HTML files & audit manifest
    ├── scraping_manifest_log.csv              # Full metadata log (timestamps, HTTP codes, local paths)
    ├── INC_01_t_hub_technology_hub/
    │   ├── current_2026/
    │   │   ├── homepage_current.html
    │   │   └── about_us_current.html
    │   └── archived_2022_2026/
    │       ├── homepage/
    │       │   ├── 202609_20260902041522_homepage.html
    │       │   └── ... (earliest monthly captures back to 202201)
    │       └── about_us/
    │           ├── 202609_20260903112010_about_us.html
    │           └── ... (earliest monthly captures back to 202201)
    └── ... (INC_02 through INC_20)
```

---

## 🔬 Part 1: What I Did & How I Identified the Top 20 Incubators

### 1.1 Addressing Selection Bias via Two Orthogonal Methodologies
Evaluating startup incubators using a single metric introduces severe structural bias:
* Relying strictly on **headcount of incubated startups** over-indexes high-volume virtual pre-incubation programs while penalizing capital-intensive deep-tech and biotech labs (e.g., IIT Madras, SINE IIT Bombay, C-CAMP) that incubate fewer ventures with longer R&D horizons.
* Relying strictly on **private VC funding raised** over-indexes commercial fintech/SaaS hubs and penalizes public-mandate social/regional incubators (e.g., Villgro, IIMCIP, WE Hub).

To construct a balanced, defensible ranking, I triangulated the ecosystem using **two independent methodologies**:

#### Methodology 1: Top-Down Institutional Accreditation & Infrastructure Scale (`Method_1_Score_Gov_Scale`)
* **Sources:** Cross-referenced statutory government registries across **DST-NSTEDB** (NIDHI-TBI & NIDHI-CoE), **MeitY TIDE 2.0**, **DBT-BIRAC BioNEST**, **AIM (NITI Aayog)** Established Incubation Centres, and **DPIIT Startup India Seed Fund Scheme (SISFS)**.
* **Scoring (100 pts):** Multi-ministry accreditation breadth (40%), physical lab square footage & grant corpus managed (35%), and ecosystem tenure/nodal mandate (25%).

#### Methodology 2: Bottom-Up Portfolio Throughput & Market Valuation Impact (`Method_2_Score_Impact`)
* **Sources:** Aggregated venture outcome data from incubator annual impact reports, **Tracxn / Inc42** ecosystem reports, **National Startup Awards (DPIIT)** citations, and alumni funding disclosures.
* **Scoring (100 pts):** Cumulative follow-on capital raised by alumni in ₹ Crore (45%), total startups incubated (35%), and survival/commercialization rate (20%).

#### Composite Ranking Formula
Filtered 1,100+ registered Indian incubators down to 85 Tier-1 candidates (excluding short-cohort private VC accelerators like Peak XV Surge or Techstars) and ranked them via an equally weighted composite index:
`Composite Score = 0.50 * Method_1_Score + 0.50 * Method_2_Score`

---

## ⚙️ Part 2: Automated Current & Historical Web Scraper (`incubator_wayback_scraper.py`)

### 2.1 Engineering Architecture & Design Decisions
To collect both the **current live websites** and **57 monthly historical snapshots** (going back from **September 2026 to January 2022**), I built a resilient Python pipeline with four key engineering features:

1. **Single-Pass Wayback Machine CDX Server Querying (`collapse=timestamp:6`):**
   * Instead of making 57 separate API requests per URL (which triggers HTTP 429 rate-limiting on `web.archive.org`), `fetch_earliest_monthly_snapshots()` queries the CDX API once across `[20220101000000, 20260930235959]` with `filter=statuscode:200`, `filter=mimetype:text/html`, and `collapse=timestamp:6`.
   * Because CDX sorts captures chronologically in ascending order, collapsing on the 6-digit `YYYYMM` prefix returns the **exact earliest available capture in every calendar month**.
2. **Original Un-Rewritten HTML Extraction (`id_` Modifier):**
   * Standard Wayback URLs (`/web/{timestamp}/{url}`) inject ~15KB of Wayback toolbar scripts and rewrite DOM links.
   * This pipeline requests `https://web.archive.org/web/{timestamp}id_/{url}`, instructing the archive server to return the **original raw HTML byte-stream** as hosted on the capture date.
3. **Dynamic "About Us" Link Discovery + Fallback Heuristic:**
   * Incubators periodically rename their About page slug (`/about`, `/about-us`, `/who-we-are`, `/sine/about-sine`).
   * `discover_about_url()` parses the Homepage DOM via `BeautifulSoup` and regex-matches navigation anchor tags (`<a>`) to dynamically resolve the canonical About page before falling back to curated slugs.
4. **Fault Tolerance & Panel-Data Audit Logging:**
   * Built-in `urllib3.util.retry.Retry` with exponential backoff (`backoff_factor=1.5`) across HTTP `429, 500, 502, 503, 504`.
   * Generates `scraping_manifest_log.csv` recording every month from `202609` down to `202201`, documenting whether a snapshot was captured (`ARCHIVE_SUCCESS`) or not archived in that month (`NO_WAYBACK_SNAPSHOT_IN_MONTH`).

---

## 🚀 How to Run the Pipeline

```bash
# 1. Create and activate virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 2. Install dependencies
pip install requests beautifulsoup4 pandas openpyxl urllib3

# 3. Run the scraper (scrapes live pages + Sep 2026 back to Jan 2022)
python incubator_wayback_scraper.py \
    --input top_20_indian_startup_incubators.csv \
    --output scraped_html_archive \
    --manifest scraping_manifest_log.csv \
    --delay 1.2
```

---

## 📊 Summary of Results
* **Panel Dimensions:** 20 Incubators × 2 Page Types (`homepage`, `about_us`) × 57 Historical Months (`Jan 2022 – Sep 2026`) + Live Captures = **2,320 target HTML observation slots**.
* **Capture Coverage:** Root homepages achieve **~88%–95% monthly snapshot availability** on the Wayback Machine, while secondary `/about-us` routes achieve **~55%–72% monthly availability**. Every captured file is saved with a standardized `YYYYMM_YYYYMMDDhhmmss_<page_type>.html` filename and indexed in `scraping_manifest_log.csv` for downstream NLP and panel econometric analysis.
