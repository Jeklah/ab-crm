# Changelog

## [Unreleased] — Codebase Cleanup & Refactor (ab-refactor branch)

This entry documents a broad clean-up pass across the repository. No application
behaviour was intentionally changed; the goals were:

- Remove code and files that are no longer needed.
- Fix dependency drift between what the code actually imports and what is declared
  in `requirements.txt`.
- Fix a latent runtime crash caused by a mismatched OpenAI client version.
- Make every changed file consistently readable (sorted imports, double-quoted
  strings, PEP-8-width lines).

---

### Deleted — one-off scripts and stale data files

The root of the repository had accumulated a collection of migration scripts,
one-time import helpers, and raw data files that were written for specific tasks
and never removed afterwards. They are not imported by the application and serve
no ongoing purpose.

**Scripts removed:**

| File | Original purpose |
|---|---|
| `budget.py` | Ad-hoc budget calculation script |
| `bulk_alt_upload.py` | One-time bulk alternative-part uploader |
| `email_analysis.py` | Stand-alone email analysis prototype |
| `import_manufacturer_approvals.py` | One-time manufacturer approvals importer |
| `import_pieces_per_pound.py` | One-time pieces-per-pound data importer |
| `jwt_generate.py` | Manual JWT token generator (dev utility) |
| `migrate_data_to_postgres.py` | One-time SQLite → PostgreSQL migration |
| `migrate_part_numbers_direct.py` | One-time part-number migration |
| `run_migration.py` | Generic migration runner |
| `split_models.py` | Dev utility for splitting the models module |
| `routes/rfqs.bk` | Uncommitted backup copy of `routes/rfqs.py` |

**Data / asset files removed:**

| File | Notes |
|---|---|
| `H120.csv`, `H125.csv`, `H135.csv`, `H160.csv` | Helicopter model part-number CSVs used by a deleted import script |
| `bk117.csv`, `dauphin.csv`, `puma.csv`, `super_puma.csv`, `cherry_alts.csv` | Same — helicopter model CSVs |
| `L030_import.xlsx` | One-time import spreadsheet |
| `insert_ppp.csv`, `sample_ppp_data.csv` | Pieces-per-pound sample / staging data |
| `skipped_rows_20260105_113335.csv` | Leftover output from a past import run |
| `country_name_mapping.json` | Static mapping superseded by the `pycountry` library |

---

### `requirements.txt` — reduced from ~190 packages to ~40

**Why it was a problem:**

The file had grown to ~190 pinned packages, most of which came from a machine-learning
/ Gradio experimentation period (`torch`, `transformers`, `accelerate`,
`bitsandbytes`, `gradio`, `huggingface-hub`, `lm-eval`, `opencv-*`, etc.). None
of these are imported anywhere in the running application. Keeping them meant:

1. Every fresh `pip install -r requirements.txt` pulled in gigabytes of ML
   libraries that do nothing.
2. Version pins for those libraries caused frequent conflicts with the packages
   that *are* used.
3. Several packages that the application **does** import were missing entirely,
   meaning the app only worked on machines that happened to have them installed
   from a previous, unrelated environment.

**What was removed:**

All ML/AI framework packages and their transitive dependencies, including but not
limited to: `torch`, `transformers`, `accelerate`, `bitsandbytes`, `peft`,
`optimum`, `datasets`, `huggingface-hub`, `gradio`, `gradio_client`,
`safetensors`, `tokenizers`, `sentencepiece`, `lm-eval`, `rouge_score`,
`sacrebleu`, `wandb`, `sentry-sdk`, `scipy`, `scikit-learn`, `numba`,
`matplotlib`, `altair`, `opencv-*`, `fastapi`, `uvicorn`, `starlette`,
`SpeechRecognition`, `pytesseract`, `tabula-py` (the full Tabula Java wrapper),
`tiktoken`, `grpcio`, `tensorboard`, and many more.

**What was added (packages that were actively used but undeclared):**

| Package | Where it is used | Why it was missing |
|---|---|---|
| `flask-login` | `app.py`, `routes/auth.py`, most route files | Simply never added |
| `flask-apscheduler` | `app.py` — scheduled jobs | Simply never added |
| `Flask-Session` | `app.py` — server-side sessions | Simply never added |
| `waitress` | `wsgi.py` — production WSGI server | Simply never added |
| `hubspot-api-client` | `routes/hubspot_integration.py` | Simply never added |
| `striprtf` | `routes/upload.py`, `routes/files.py`, `routes/parts_list.py` | Simply never added |
| `pypdf` | `routes/offers.py` — `PdfReader` | Simply never added |
| `PyJWT` | `jwt_generate.py` (now deleted); kept for any future JWT use | Simply never added |
| `watchdog` | `folder_watcher.py` | Simply never added |

**OpenAI version fix (critical):**

`openai` was pinned to `0.28.0`, which uses the old procedural API
(`openai.ChatCompletion.create(…)`). The codebase had already been partially
migrated to the new v1.x client style (`from openai import OpenAI; client =
OpenAI()`). This mismatch caused a silent runtime incompatibility in
`generate_industry_insights` (see `ai_helper.py` below). The pin has been changed
to `openai>=1.0.0`.

**Structure:**

The file is now grouped by concern with inline comments explaining why each
package is present, making future audits straightforward.

---

### `ai_helper.py` — OpenAI client consistency fix + import cleanup

**The bug:**

`generate_industry_insights` was the one remaining function still using the old
v0.x API call style:

```python
# Before — v0.x style, broken against openai>=1.0.0
response = openai.ChatCompletion.create(model="gpt-4o", …)
response_content = response.choices[0].message['content'].strip()
```

With `openai>=1.0.0` this raises `AttributeError: module 'openai' has no
attribute 'ChatCompletion'`. It was fixed to match every other function in the
same file:

```python
# After — v1.x style
response = client.chat.completions.create(model="gpt-4o", …)
response_content = response.choices[0].message.content.strip()
```

Note that `response.choices[0].message` is now a typed `ChatCompletionMessage`
object, so its `.content` attribute is accessed as a property rather than a dict
key.

**Import changes:**

- Removed the redundant bare `import openai` (only kept as a comment-noted
  reference for `AuthenticationError` / `APIError`; the actual client is
  instantiated via `from openai import OpenAI`).
- `client = openai.Client(…)` → `client = OpenAI(…)` — the `openai.Client`
  alias was removed in v1.x.
- Stdlib imports sorted and grouped above third-party imports.

---

### `app.py` — import organisation

No logic was changed. The import block was reorganised:

- stdlib imports grouped together at the top.
- Third-party imports (`flask`, `flask_*`, `markdown`) in a second group.
- Local imports (`db`, `models`, `routes.*`) in a third group, sorted
  alphabetically.
- Removed unused stdlib imports that were left over from an older version of the
  file (`import email`, `import imaplib`, and several `from email.*` imports that
  are only needed inside individual route modules).
- All string literals normalised to double quotes for consistency with the rest of
  the refactor.

---

### `models/part_2.py` — import deduplication and cleanup

The file had a number of import hygiene issues that were fixed:

- `import datetime` and `from datetime import datetime` both appeared; the bare
  `import datetime` was redundant and removed.
- `import os` and `import os as _os` both appeared; collapsed to a single
  `import os as _os` (the alias is used for `DATABASE_URL` to avoid shadowing).
- All imports sorted and grouped (stdlib → third-party → local).
- String quotes normalised to double quotes.

---

### Route files — import line reformatting

The following route files had their imports restructured. In every case the change
is purely cosmetic — no symbols were added or removed, and no logic was altered:

| File | What changed |
|---|---|
| `routes/customers.py` | Single 300-character `from models import …` line expanded to one-symbol-per-line, grouped by origin |
| `routes/rfqs.py` | Same treatment; also removed unused `from flask import Flask` |
| `routes/dashboard.py` | Import grouping and sort |
| `routes/handson.py` | Import grouping and sort |
| `routes/offers.py` | Import grouping and sort |
| `routes/sales_orders.py` | Import grouping and sort |
| `routes/stock_movements.py` | Import grouping and sort |

**Why this matters:** Python import errors (circular imports, missing packages)
are reported at the line level. When twenty symbols are crammed onto one line it
is impossible to tell which one caused the error. The expanded format makes
debugging straightforward.

---

### Moved to `scripts/` and `data/`

Utility scripts and supporting data files that are still occasionally useful (but
should not live in the application root) have been moved into dedicated
subdirectories:

- `scripts/` — any helper scripts retained for operational use.
- `data/` — any reference data files retained for lookups or re-imports.

These directories are untracked (`data/` and `scripts/` appear in the untracked
list in `git status`) and should be added to `.gitignore` if they contain
sensitive or environment-specific content.

---

### How to reinstall dependencies after this change

If you are working in an existing virtual environment, the safest approach is to
recreate it from scratch to avoid leftover packages from the old `requirements.txt`
causing unexpected behaviour:

```
deactivate
rm -rf venv
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
```
