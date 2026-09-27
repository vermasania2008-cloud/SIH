# Airfare Index for India

A compliance-aware, route-weighted airfare indicator designed as a beta-statistics prototype for SIH26056 and the MoSPI/DIID context. This project is not an official CPI release, but it demonstrates a transparent and reproducible framework for measuring fare movement responsibly, with explicit source labeling, governance guardrails, and statistically explainable methods.

## Why the Project Matters

A raw airfare average is not a reliable signal of inflation. It can be distorted by:

- route composition shifts
- sold-out low-fare inventory
- changing booking horizons
- seasonal travel demand
- inconsistent source quality

The project addresses this challenge by combining normalized fare observations, route-weighted aggregation, and a GEKS-style transitive comparison framework. The result is a decision-support system that is explainable, auditable for a prototype, and ready to absorb an approved production feed when institutional data access becomes available.

## What It Does

- Collects a small live snapshot from a public flight-search webpage when available.
- Falls back to a labeled reference CSV when the source is unavailable.
- Normalizes observations into a single fare schema.
- Calculates route-level coverage, integrity, and weighted index moves.
- Displays route trends and booking-horizon elasticity.
- Supports daily, weekly, and monthly aggregation views.
- Includes a route-by-horizon fare heatmap.
- Exposes local JSON endpoints for health, summary, frequency, and source data.
- Demonstrates Jevons and GEKS-style index concepts.
- Includes governance, compliance, and production-readiness guidance.
- Provides a polished six-page presentation PDF and a speaker-ready script.
- Uses a rolling 13-period GEKS-Jevons window with mean splicing so new periods
    do not silently rewrite the published history.
- Publishes a TPD sensitivity estimator and an unmatched arithmetic-mean negative
    control on the same matched-cell panel.
- Defines the 50-route demonstration frame, seven fixed horizons, and explicit
    route strata in the sampling metadata.

## Final Product Snapshot

This project is structured as a ready-to-present demo package for judges and stakeholders:

- executive dashboard with KPI, coverage, and integrity views
- methodology and validation pages with transitive comparison logic
- governance-led compliance framing and source-status visibility
- modular data ingest, cleaning, and comparison engine
- clear roadmap toward an official route basket and approved source feed
- six-page presentation deck designed for concise and professional stakeholder communication

### SIH Judge Resources

- [Beginner SIH problem and requirements explanation](SIH_PROBLEM_AND_REQUIREMENTS_SIMPLE.txt)
- [Beginner-friendly project guide PDF](presentation/sih_airfare_project_guide.pdf)
- [Judge cheat sheet](JUDGE_BRIEF.txt)
- [Updated team presentation script](presentation/team_script.txt)
- [PDF guide generator](presentation/create_sih_guide_pdf.py)

## Run Locally

```powershell
py -3 -m pip install -r requirements.txt
streamlit run app.py
```

Open the local URL printed by Streamlit, normally `http://localhost:8501`.

To start FastAPI and Streamlit together on Windows:

```powershell
.\start_project.ps1
```

This starts FastAPI at `http://127.0.0.1:8000`, starts Streamlit at
`http://localhost:8501`, and stops the FastAPI process when the dashboard is
closed.

## Workflow

```mermaid
flowchart LR
    A[Public fare webpage or permitted feed] --> B[Source adapter]
    B --> C[Normalized fare rows]
    C --> D[Quality checks and status]
    D --> E[Route-weighted metrics]
    E --> F[Streamlit dashboard]
    D --> G[Validation and governance views]
    B -. unavailable .-> H[Reference CSV fallback]
    H --> C
```

### 1. Application startup

[app.py](app.py) configures Streamlit, injects the theme, loads data into `st.session_state`, shows the source-status label, and routes the sidebar selection to one of four views.

### 2. Data acquisition

[core/ingestion.py](core/ingestion.py) is the entry point. It first checks optional environment-based API configuration, then calls the public-source path in [core/data_source.py](core/data_source.py).

The public-source adapter currently requests a small set of routes and extracts rupee-formatted values from the returned HTML. It records the date, route, departure date, horizon, fare, and scrape status. It does not bypass CAPTCHA or anti-bot controls.

The public webpage snapshot is disabled by default so the dashboard opens quickly and predictably. Enable it only for an approved, permitted environment with `AIRFARE_ENABLE_PUBLIC_SNAPSHOT=true`. The Amadeus API path remains the preferred live source when credentials are configured.

If no usable live rows are returned, [data/airfare_reference.csv](data/airfare_reference.csv) is loaded as the fallback. The sidebar makes this visible as either `LIVE PUBLIC WEB SNAPSHOT` or `REFERENCE CSV FALLBACK`.

### 3. Cleaning and validation

[core/cleaning.py](core/cleaning.py) contains the reusable cleaning rules for fare bounds, horizon padding, log-scale outliers, failed scrapes, and imputation flags.

[core/lineage.py](core/lineage.py) builds deterministic `raw_quote`, `observation`, `cell`, and `cell_price` tables. Rejected fare rows remain in the observation layer with a reason code rather than disappearing silently.

The ingestion path persists the raw snapshot before cleaning, then materializes the cleaned index and lineage artifacts. The current prototype still needs the official sampling frame and richer fare-product identifiers for full production compliance.

### 4. Index and quality metrics

[core/metrics.py](core/metrics.py) calculates the route-weighted fare indicator and an explicit quality summary containing observation count, configured route coverage, fare completeness, and pipeline integrity. Empty data is reported as zero integrity rather than silently passing a readiness check.

[core/formulas.py](core/formulas.py) contains the elementary Jevons calculation, a missing-aware GEKS-Jevons calculation, and rolling mean-splice logic. Missing bilateral comparisons remain missing rather than becoming neutral links. The index engine uses all valid quotes in a cell, a 13-period GEKS window, an unmatched arithmetic-mean comparison, and observation-count-weighted TPD sensitivity.

[core/api.py](core/api.py) creates a compact JSON-friendly summary with a quality block containing coverage, completeness, integrity, and vintage metadata.

[core/materialization.py](core/materialization.py) writes a local index vintage artifact with both supported time axes, route weights, methodology, quality metadata, and lineage tables.

[api_server.py](api_server.py) provides the FastAPI service for live offers and index consumers:

```powershell
py -3 -m pip install -r requirements.txt
py -3 api_server.py
```

The API is FastAPI-based and exposes interactive documentation at `http://127.0.0.1:8000/docs`.

Available endpoints are `GET /health`, `GET /sources`, `GET /readiness`, `GET /flights`, `GET /live-data`, `GET /summary`, `GET /index`, `GET /frequencies`, `GET /data`, `GET /vintage`, `GET /revisions`, `GET /sdmx`, and `GET /lineage/{table_name}`. `/sources` shows which live, approved, public, and demo paths are configured. `/live-data` fetches current Amadeus offers and returns normalized fare observations, including fare basis, RBD, cabin, taxes, brand, and flight identity when supplied; it fails with `503` when live credentials or provider access are unavailable and never falls back to the reference CSV. The summary, index, vintage, revisions, and SDMX endpoints read published artifacts. `/lineage/{table_name}` exposes stored raw quotes, observations, cells, or cell prices. `/data` is the demo-friendly source endpoint and may return the clearly labeled fallback dataset.

The server binds to `127.0.0.1` by default. To share it on a trusted local network, set `AIRFARE_API_HOST=0.0.0.0` before running it. Do not expose this prototype directly to the public internet.

To request current live observations from the FastAPI service after configuring Amadeus credentials:

```powershell
Invoke-RestMethod "http://127.0.0.1:8000/live-data?departure_date=2026-10-01&adults=1&currency=INR"
```

This endpoint returns only provider data. Without valid live credentials or provider access it returns `503`; it never substitutes the reference CSV.

### 5. Dashboard views

- **Executive Summary:** indicator, coverage, integrity, problem statement, and methodology positioning.
- **Index Trends & Dashboards:** route trend lines and booking-horizon elasticity.
- **Index Trends & Dashboards:** daily/weekly/monthly selector, route × horizon heatmap, estimator comparison, and quality ledger.
- **Methodology & Validation:** chain-drift demonstration and statistical explanation.
- **Compliance & Governance:** source policy, rate limits, CAPTCHA restrictions, and future deployment direction.

## Data Schema

The normalized row format is:

| Column | Meaning |
|---|---|
| `scrape_date` | Date the fare observation was collected |
| `departure_date` | Intended travel date |
| `route` | City-pair code such as `DEL-BOM` |
| `carrier` | Carrier or source label |
| `booking_horizon` | Advance purchase window such as `T+7` |
| `base_fare` | Parsed fare value in INR before future tax decomposition |
| `scrape_status` | `SUCCESS` or failure state |

## Optional Data Configuration

For a permitted CSV feed with the same schema:

```powershell
$env:AIRFARE_LIVE_CSV = "C:\path\to\approved_fare_feed.csv"
streamlit run app.py
```

An approved JSON/API integration can use:

```powershell
$env:AIRFARE_API_URL = "https://approved-source.example/fare-feed"
$env:AIRFARE_API_KEY = "your-secret-key"
```

Do not commit credentials. Do not use API keys or source access that your team is not authorized to use.

### Provider Configuration Checklist

Add your provider keys in a local shell or secret manager before starting the services:

For local development, copy `.env.example` to `.env` and fill the authorized
provider variables. The API and Streamlit app load `.env` automatically. Keep
`.env` out of version control and never paste real keys into `.env.example`.

```powershell
# Preferred live provider
$env:AMADEUS_CLIENT_ID = "your-client-id"
$env:AMADEUS_CLIENT_SECRET = "your-client-secret"
$env:AMADEUS_ENV = "test"

# Optional fallback / partner names accepted by the project
$env:APIX_AMADEUS_KEY = "your-client-id"
$env:APIX_AMADEUS_SECRET = "your-client-secret"
$env:APIX_AMADEUS_ENV = "test"

$env:AIRFARE_LIVE_CSV = "C:\path\to\approved_fare_feed.csv"
$env:AIRFARE_ENABLE_PUBLIC_SNAPSHOT = "false"

$env:SPICEJET_API_KEY = "your-spicejet-partner-key"
$env:SPICEJET_API_BASE_URL = "https://partner.example.com/api"
$env:SPICEJET_PARTNER_ACCOUNT = "your-partner-account"

$env:TRAVELPAYOUTS_API_KEY = "your-key"
$env:AVIATIONSTACK_API_KEY = "your-key"
```

Use the same pattern in your local environment or in a `.env` file that is loaded by your shell before launching the app. The registry in [core/source_registry.py](core/source_registry.py) shows which providers are recognized and which variables are expected.

### Live Flight API Configuration

Important current status: Amadeus decommissioned its self-service developer portal on July 17, 2026. New users may not be able to create self-service applications or obtain Client ID/Secret credentials through the old signup flow. The current website routes new access requests through the Enterprise API process.

If Amadeus has issued your organization credentials through Enterprise access, configure them in the shell before starting the API. Use the test environment while developing, then switch to production only after fare access is approved:

Copy `.env.example` as a local reference, but keep the actual credentials in environment variables or a secret manager.

```powershell
$env:AMADEUS_CLIENT_ID = "your-client-id"
$env:AMADEUS_CLIENT_SECRET = "your-client-secret"
$env:AMADEUS_ENV = "test"
py -3 api_server.py
```

Credentials from the companion `Hackathon-main` project are also accepted:

```powershell
$env:APIX_AMADEUS_KEY = "your-client-id"
$env:APIX_AMADEUS_SECRET = "your-client-secret"
$env:APIX_AMADEUS_ENV = "test"
py -3 api_server.py
```

Example live fare request:

```text
http://127.0.0.1:8000/flights?origin=DEL&destination=BOM&departure_date=2026-10-01&adults=1&currency=INR
```

Without Amadeus credentials, `/flights` returns a configuration error. It never labels the reference CSV as a live flight price.

If you cannot obtain Amadeus access, use an approved normalized CSV feed through `AIRFARE_LIVE_CSV` or integrate another licensed provider adapter. Do not claim the fallback CSV is live.

## Project Structure

```text
app.py                         Streamlit entry point and router
core/data_source.py            Public source adapter and CSV fallback
core/ingestion.py              Data acquisition orchestration
core/cleaning.py               Fare cleaning and imputation rules
core/formulas.py               Jevons, GEKS, and splice calculations
core/metrics.py                Route indicator and quality score
core/api.py                    JSON-friendly summary contract
core/lineage.py                Raw, observation, cell, and cell-price tables
core/materialization.py        Persisted index vintage and lineage artifacts
data/airfare_reference.csv     Stable offline fallback dataset
data/materialized/             Latest local vintage and lineage exports
views/                          Dashboard pages
assets/styles/style.css         Theme, blurred background, glassmorphism
images/bg.jpeg                 Dashboard background image
presentation/                  PDF deck, script, and PDF generator
tests/                         Automated tests
```

## Testing

```powershell
py -3 -m pytest -q
```

The current test suite covers route-weighted indicator behavior, estimator benchmarks, API readiness, and pipeline-quality reporting.

## Honest Limitations

1. The public webpage parser is a prototype adapter. A live HTTP response does not automatically prove that every parsed rupee value is a verified fare card.
2. The reference CSV is fallback/demo data and must never be presented as live market data.
3. The route weights in the prototype are demonstration values. Production must consume the official PSD basket and weights.
4. The system does not bypass CAPTCHA, robots.txt restrictions, rate limits, or terms of service.
5. The index is a beta statistic and is not an official MoSPI CPI release.
6. The prototype now materializes lineage and vintages, but it does not yet implement the full 50-route stratified PPS frame, seven-horizon collection design, official fare-basis identifiers, or complete revision workflow from the requirement PDF.
7. The dashboard reads the materialized observation layer; the production target remains a separately deployed API-only UI with authentication and network controls.

## Presentation Materials

- [Presentation PDF](presentation/airfare_index_presentation.pdf)
- [Speaker script](presentation/team_script.txt)
- Live screenshots used in the deck: `presentation/executive_summary.png`, `presentation/analytics_heatmap.png`, `presentation/validation.png`, and `presentation/governance.png`.

The deck is intentionally kept to six pages for a concise, professional presentation. Regenerate it with:

```powershell
py -3 presentation\create_presentation_pdf.py
```

## Recommended Next Production Steps

1. Obtain an approved source or data-sharing agreement.
2. Replace HTML regex extraction with a source-specific parser for documented fare basis or RBD fields.
3. Implement the 50-route stratified PPS frame and all seven fixed horizons.
4. Load the official route basket and weights from MoSPI/PSD.
5. Add scheduled revision vintages with provisional, revised, and final status.
6. Back-test on at least 30 days of approved observations and publish the full quality block.
