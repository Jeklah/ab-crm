# Aerospace Hardware CRM

A comprehensive Customer Relationship Management system built for an aerospace hardware distributor. The system manages the full sales and procurement lifecycle — from initial customer contact and RFQ intake through supplier quoting, costing, order fulfilment, and finance — with deep email integration and AI-assisted workflows throughout.

---

## Table of Contents

- [Overview](#overview)
- [Key Modules](#key-modules)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Linux](#linux)
  - [Windows](#windows)
- [Configuration](#configuration)
- [Database Setup](#database-setup)
- [Running the Application](#running-the-application)
- [Project Structure](#project-structure)
- [Integrations](#integrations)
- [Migrations](#migrations)
- [Development Notes](#development-notes)

---

## Overview

This CRM is purpose-built for a company that supplies approved, traceable fasteners, consumables, and hardware to operators, OEMs, and MROs in the aerospace sector. It replaces a collection of spreadsheets and disconnected tools with a single web-based platform that every member of the team uses daily.

Core capabilities at a glance:

- Full customer and contact lifecycle management with activity tracking and call lists
- AI-powered parts list intake — extracts part numbers and quantities from emails, PDFs, and Excel files automatically
- Multi-supplier quoting and side-by-side costing with stock availability checks
- Microsoft Outlook integration via the Graph API for two-way email sync
- Sales order and purchase order management
- Inventory, excess stock, and stock movement tracking
- A supplier-facing portal for quote submission
- A customer-facing quoting portal
- A support ticketing module
- Finance ledger and invoice management
- Manufacturer approvals (QPL) database

---

## Key Modules

### Sales CRM (`routes/customers.py`, `routes/salespeople.py`)
- Hierarchical customer records with parent/child company relationships
- Contact management with full communication history
- Call lists with snooze, priority, and status tracking
- Monthly sales planner with AI-generated target suggestions
- Customer churn risk assessment based on purchase patterns
- Automated customer news collection via Perplexity AI
- Bulk contact operations and segmentation by tag, region, or company type
- Geographic deep-dive analysis (`routes/geo_deepdive.py`)

### Parts List & Quoting (`routes/parts_list.py`, `routes/parts_list_ai.py`)
- AI extraction of part numbers and quantities from any input format
- Multi-supplier quote campaigns with email tracking and automated follow-up
- Visual costing interface with stock, BOM, and supplier comparison
- Availability checking across VQ, PO, stock, excess, and ILS data sources
- Monroe / QPL manufacturer approval lookups
- Common parts reporting across multiple customers
- Excel and CSV export for procurement handoff

### RFQs & Offers (`routes/rfqs.py`, `routes/offers.py`)
- Inbound RFQ management from customer emails or manual entry
- AI-assisted quote parsing from supplier PDF and email responses
- Offer creation with margin analysis and currency handling
- PDF quote generation via ReportLab / wkhtmltopdf

### Orders (`routes/sales_orders.py`, `routes/purchase_orders.py`)
- Full sales order lifecycle with line-level status tracking
- Purchase order generation linked to sales order lines
- Expediting module for chasing overdue PO lines
- SO import from Excel
- Sales order acknowledgment PDF generation

### Email Integration (`routes/emails.py`, `routes/email_signatures.py`)
- Full two-way Outlook sync via Microsoft Graph API
- Automatic contact matching and communication logging
- AI email triage — categorises inbound emails and can auto-create parts lists or RFQ records
- Per-user email signatures with image support
- Bulk email campaigns with open/reply tracking
- Folder rules for automated inbox processing

### Supplier Portal (`routes/supplier_portal.py`, `routes/portal_api.py`)
- Suppliers log in and submit quotes directly against open RFQ lines
- Quote lines flow automatically into the internal costing view

### Customer Quoting Portal (`routes/customer_quoting.py`, `routes/cqs.py`, `routes/vqs.py`)
- Customers can receive and respond to quotes via a branded portal

### Finance (`routes/finance.py`, `routes/invoices.py`)
- Chart of accounts, journal entries, and account reconciliation
- Invoice creation and tracking
- Multi-currency support with live exchange rate fetching

### Inventory (`routes/stock_movements.py`, `routes/excess.py`)
- Stock movement logging (inbound, outbound, adjustments)
- Excess stock list management with customer matching

### Ticketing (`routes/tickets.py`)
- Internal and external support ticket management
- Workspace-based routing with notification preferences
- External user access for customer-submitted tickets

### Other Modules
| Module | Description |
|---|---|
| `routes/manufacturers.py` | Manufacturer records and approval database |
| `routes/manufacturer_approvals.py` | QPL import and lookup |
| `routes/parts.py` | Part number master data |
| `routes/price_lists.py` | Supplier price list management |
| `routes/projects.py` | Project-based BOM and parts list tracking |
| `routes/bom.py` | Bill of materials management |
| `routes/ils.py` | ILS (Industry Lookup Service) integration |
| `routes/marketplace.py` | Airbus marketplace export |
| `routes/partsbase.py` | PartsBase listing integration |
| `routes/nexar.py` | Nexar component data lookup |
| `routes/hubspot_integration.py` | HubSpot CRM sync |
| `routes/team_tracker.py` | Team activity and performance tracking |
| `routes/salesperson_metrics.py` | Individual salesperson KPI dashboards |
| `routes/admin.py` | User management and system settings |
| `routes/currencies.py` | Currency and exchange rate management |
| `routes/tax_rates.py` | Tax rate configuration |
| `routes/imports.py` | Bulk data import tooling |
| `routes/handson.py` | Handsontable-powered spreadsheet editor views |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Web framework | Flask 3.0 |
| Database | PostgreSQL (via psycopg2) |
| ORM / introspection | SQLAlchemy 2.0 (schema introspection only) |
| WSGI server | Waitress (Windows & Linux) |
| AI | OpenAI GPT-4o |
| Email | Microsoft Graph API (MSAL) |
| CRM sync | HubSpot API |
| PDF generation | pdfkit + wkhtmltopdf, ReportLab |
| PDF extraction | pdfplumber, pypdf, tabula, pdf2image |
| Spreadsheets | openpyxl, pandas |
| Frontend | Bootstrap, Handsontable, vanilla JS / AJAX |
| Background jobs | Flask-APScheduler |
| Sessions | Flask-Session (filesystem) |
| Auth | Flask-Login |
| File watching | watchdog |

---

## Prerequisites

**All platforms:**
- Python 3.10 or later
- PostgreSQL 14 or later
- `wkhtmltopdf` (for PDF generation)

**Linux (Debian/Ubuntu):**
```bash
sudo apt update
sudo apt install postgresql wkhtmltopdf
```

**Windows:**
- [PostgreSQL installer](https://www.postgresql.org/download/windows/)
- [wkhtmltopdf installer](https://wkhtmltopdf.org/downloads.html) — install to the default path (`C:\Program Files\wkhtmltopdf\`) or set `WKHTMLTOPDF_PATH` in your `.env`
- `pywin32` is required for Outlook COM automation (installed automatically via `requirements.txt`)

---

## Installation

### Linux

```bash
# 1. Clone the repository
git clone <repo-url>
cd crm

# 2. Create and activate a virtual environment
python3 -m venv venv
source venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Copy and fill in the environment file
cp .env.example .env
# Edit .env — see Configuration section below

# 5. Set up the database
psql $DATABASE_URL -f schema.sql

# 6. Apply any outstanding migrations
for f in migrations/*.sql; do psql $DATABASE_URL -f "$f"; done

# 7. Start the server
python wsgi.py
```

### Windows

```bat
REM 1. Clone the repository and open a command prompt in the project folder

REM 2. Create and activate a virtual environment
python -m venv venv
venv\Scripts\activate

REM 3. Install dependencies
pip install -r requirements.txt

REM 4. Copy and fill in the environment file
copy .env.example .env
REM Edit .env — see Configuration section below

REM 5. Set up the database (requires psql on PATH)
psql %DATABASE_URL% -f schema.sql

REM 6. Start the server (double-click or run from cmd)
start.bat
```

`start.bat` checks for Python and Waitress, then runs `wsgi.py` which starts Waitress on `http://0.0.0.0:8080`.

---

## Configuration

Create a `.env` file in the project root. All keys are read at startup via `python-dotenv`.

```ini
# ── Database ──────────────────────────────────────────────────────────────────
DATABASE_URL=postgresql://user:password@localhost:5432/crm

# ── Flask ─────────────────────────────────────────────────────────────────────
SECRET_KEY=change-me-to-a-long-random-string
API_KEY=internal-api-key-for-external-agents

# ── OpenAI ────────────────────────────────────────────────────────────────────
OPENAI_API_KEY=sk-...

# ── Microsoft Graph API (Outlook email sync) ──────────────────────────────────
# Register an app in Azure Active Directory and grant Mail.ReadWrite + Contacts.Read
AZURE_CLIENT_ID=...
AZURE_CLIENT_SECRET=...
AZURE_TENANT_ID=...

# ── HubSpot (optional) ────────────────────────────────────────────────────────
HUBSPOT_API_KEY=...

# ── Apollo.io (optional — customer enrichment) ────────────────────────────────
APOLLO_API_KEY=...

# ── Exchange rates ────────────────────────────────────────────────────────────
EXCHANGE_RATE_API_KEY=...

# ── Ticketing hub (optional — external ticket sync) ───────────────────────────
TICKETS_HUB_URL=
TICKETS_HUB_API_KEY=
TICKETS_BASE_URL=

# ── Email (IMAP fallback — used if Graph API is not configured) ───────────────
EMAIL_HOST=
EMAIL_PORT=993
EMAIL_USER=
EMAIL_PASSWORD=

# ── PDF generation ────────────────────────────────────────────────────────────
# Only needed if wkhtmltopdf is not on PATH or not in the default Windows location.
# WKHTMLTOPDF_PATH=/usr/local/bin/wkhtmltopdf
```

> **Security:** never commit `.env` to version control. It is listed in `.gitignore`.

---

## Database Setup

The database schema is managed with two artefacts:

| File | Purpose |
|---|---|
| `schema.sql` | Full PostgreSQL schema — run once on a fresh database |
| `migrations/*.sql` | Incremental changes applied in filename order |

**Fresh install:**
```bash
psql $DATABASE_URL -f schema.sql
```

**Applying migrations after a pull:**
```bash
# Linux / macOS
for f in migrations/*.sql; do
    echo "Applying $f..."
    psql $DATABASE_URL -f "$f"
done

# Windows (PowerShell)
Get-ChildItem migrations\*.sql | Sort-Object Name | ForEach-Object {
    Write-Host "Applying $($_.Name)..."
    psql $env:DATABASE_URL -f $_.FullName
}
```

Migrations are plain SQL files named `YYYYMMDD_description.sql`. They are not tracked as applied — run them in order and skip any that have already been applied to your database.

---

## Running the Application

### Development
```bash
# Uses Flask's built-in server with auto-reload
flask --app app run --debug --port 5000
```

### Production (Waitress)
```bash
# Linux
python wsgi.py

# Windows
start.bat
# or directly:
python wsgi.py
```

Waitress listens on `http://0.0.0.0:8080` with 8 worker threads by default. Adjust `threads`, `connection_limit`, and `channel_timeout` in `wsgi.py` to suit your load.

The application logs to stdout at `INFO` level. Redirect to a file or a log aggregator as needed for production deployments.

---

## Project Structure

```
crm/
├── app.py                   # Flask application factory and route registration
├── wsgi.py                  # Waitress WSGI entry point
├── start.bat                # Windows one-click launcher
├── db.py                    # PostgreSQL connection pool and query helpers
├── models.py                # Top-level model re-exports
├── models/
│   ├── part_1.py            # Customer, contact, user models
│   ├── part_2.py            # Sales orders, purchase orders, PDF generation
│   ├── part_3.py            # Contacts list, communications, planner
│   ├── part_4.py            # Tags, suppliers, currencies
│   └── part_5.py            # Parts, manufacturers, stock
├── routes/                  # One Blueprint per feature area (55 files)
├── templates/               # Jinja2 HTML templates
├── static/                  # CSS, JavaScript, images
├── integrations/
│   ├── partsbase_client.py  # PartsBase API client
│   └── mirakl/              # Mirakl marketplace integration
├── migrations/              # Incremental SQL migration files
├── schema.sql               # Full PostgreSQL schema
├── ai_helper.py             # OpenAI client and prompt helpers
├── utils.py                 # Shared utility functions
├── folder_watcher.py        # Filesystem watcher for auto-import
├── data/
│   └── country_name_mapping.json   # ISO country code → name mapping
├── docs/                    # Integration guides and reference docs
├── scripts/                 # Operational helper scripts (not part of the app)
├── requirements.txt         # Python dependencies
└── .env                     # Local environment variables (not committed)
```

---

## Integrations

### Microsoft Graph API / Outlook
Used for full two-way email synchronisation. Inbound emails are matched to existing customers and contacts and logged automatically. Outbound emails are sent via Graph so they appear in the user's Sent Items.

Setup requires an Azure AD app registration with the following delegated or application permissions:
- `Mail.ReadWrite`
- `Mail.Send`
- `Contacts.Read`
- `User.Read`

Set `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, and `AZURE_TENANT_ID` in `.env`.

### OpenAI
Used for:
- Extracting part numbers and quantities from unstructured text
- Parsing supplier quotes from PDF/email content
- Generating customer outreach email suggestions
- Enriching customer records with company data
- Industry insights and sales suggestions

Requires `OPENAI_API_KEY` in `.env`. The application uses the `gpt-4o` model throughout.

### HubSpot (optional)
Bi-directional sync of customer and contact records. Set `HUBSPOT_API_KEY` in `.env`. If not configured, HubSpot features are hidden but the rest of the application is unaffected.

### Perplexity AI (optional)
Used for real-time customer news research in the sales planner. Configured via its own API key stored in the database settings page rather than `.env`.

### PartsBase
Parts listing integration for publishing stock to the PartsBase marketplace. Configured via the Settings page in the application.

### Airbus Marketplace
Export tooling for the Airbus Skywise / marketplace platform (`airbus_marketplace_export.py`, `routes/marketplace.py`).

### Nexar
Component data and availability lookup via the Nexar API (`routes/nexar.py`). API credentials are configured in the application settings.

---

## Migrations

All schema changes after the initial `schema.sql` are captured as individual SQL files in `migrations/`. Files are named `YYYYMMDD_short_description.sql` so they sort chronologically.

To add a migration:
1. Create a new file: `migrations/YYYYMMDD_what_you_changed.sql`
2. Write idempotent SQL where possible (e.g. `ADD COLUMN IF NOT EXISTS`)
3. Apply it to your local database and confirm it works before committing

There is no migration runner tracking table — the convention is that each developer keeps their local database current and migrations are applied to production manually during a deployment.

---

## Development Notes

### Platform compatibility
The application runs on both **Linux** and **Windows**. A few points to keep in mind:

- `pywin32` (Outlook COM automation in `sales_orders.py`) is only imported inside a `try/except ImportError` block, so the app starts cleanly on Linux. The feature is a no-op on non-Windows hosts.
- `wkhtmltopdf` path resolution checks `WKHTMLTOPDF_PATH` first, then falls back to the Windows default install path when `os.name == "nt"`, and otherwise relies on the system `PATH` (normal for Linux).
- All temporary files use `tempfile.gettempdir()` rather than hardcoded `/tmp/` paths.

### Adding a new route module
1. Create `routes/your_module.py` with a Flask `Blueprint`.
2. Import and register the blueprint in `app.py` (follow the existing alphabetical pattern).
3. Add any new tables to a new migration file in `migrations/`.

### Environment variables
All secrets and environment-specific settings must go in `.env`. The `.env` file is loaded at the very top of `app.py` before any other imports, so all `os.getenv()` calls throughout the application will see the values.

### Session storage
Flask-Session stores session data on the filesystem in `./flask_session/` (relative to the working directory when the server starts). This directory is created automatically and is listed in `.gitignore`.

### Background jobs
APScheduler runs several periodic tasks registered in `app.py`:
- Email synchronisation with the Graph API mailbox
- Customer news collection
- Scheduled notification delivery

Jobs run in-process — there is no separate worker process required.

### Database connection pool
`db.py` maintains a `ThreadedConnectionPool` with a maximum of 30 connections. The pool size can be adjusted in `_get_postgres_pool()` if you see connection exhaustion under high load.