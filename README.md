# From Folders to CRM: Extracting 15 Years of Client Data with AI

Helping a small equipment business build a client database from scratch. 15 years of operations, clients worldwide, zero CRM — all client info lived in folder structures of quotes, invoices, and emails. The goal: prepare structured client data for CRM migration.

**Input:** ~5,000 PDFs across 200+ client folders + 46,000 email exports (plain text)
**Output:** 935 structured client contacts, 63% enriched with email/phone/mobile
**Time:** ~10 hours active work (vs ~170 hours estimated manually at 2 min/PDF)
**Stack:** Python, PyMuPDF, Claude Sonnet/Opus, Claude Code, regex

## The Problem

The source data spanned 2013-2025, in French and English, across 5+ document template eras. The PDFs and emails included quotes, invoices, and proforma invoices — but also supplier documents, airline receipts, customs forms, and forwarded emails mixed into the same folders.

The goal: extract company name, contact person, postal address, phone, mobile, and email for every client into a clean CSV.

## Pipeline

### Phase 1: PDF Extraction

```
Find PDFs → Filter by path → Extract text → Classify document → Extract address block → Parse fields → Validate → Deduplicate → Write CSV
```

**Document classification** rejects non-outgoing documents using ~50 patterns: third-party invoices, incoming purchase orders, tax forms, customs declarations, logistics receipts, forwarded emails, internal documents.

**Address block extraction** uses 3 fallback strategies:
1. Collect non-boilerplate lines from top of first page
2. Search for attention patterns (`à l'att.`, `attn`) anywhere on first page
3. Find sender line, collect client info from lines after it
4. Search for company suffix lines (SARL, Ltd, Inc) with surrounding context

**Field classification** assigns each extracted line a type: company, person, street, postal code, city, country, or unknown. Handles edge cases like branch office names misclassified as people, cities misclassified as people, and freelancers without company names.

### Phase 2: Email Enrichment

PDFs provided names and addresses but most rows were missing email/phone/mobile. Three approaches were built and tested:

**Approach 1 — Indexed pipeline:**
Parse all emails into structured indexes (by domain, person name, PDF attachment name). Match CSV rows against indexes. Send candidates to Claude Sonnet for extraction, QA with Opus. ~800 lines. Fragile — 36% of emails were affected by a multi-line header parsing bug alone.

**Approach 2 — Regex extraction:**
Pure regex extraction from pre-matched candidate emails. No LLM calls, runs in seconds. Moved enrichment from 249 to 593 rows.

**Approach 3 — Grep-based:**
For each row missing contact info, grep the entire email archive for the person/company name, then extract from matching files. ~250 lines. Equivalent results to approach 1 with no indexing bugs.

## Results

| Metric | Count |
|--------|-------|
| PDFs processed | ~5,000 |
| Contacts extracted | 935 |
| Enriched with email/phone/mobile | 593 (63%) |
| Post-2020 enrichment rate | ~90% |
| Pre-2020 enrichment rate | ~25% |
| Extraction rounds | 9 |
| Enrichment rounds | 9 |

Pre-2020 rate is low because the email archive starts mid-2022.

## Architecture

```
                                    PDF Extraction
                                    ─────────────
Clients/                         ┌─────────────────┐
├── Client A/                    │  Find PDFs       │
│   ├── quote_2024.pdf    ──────>│  Classify docs   │──────> client_contacts.csv
│   └── invoice_2023.pdf         │  Extract address  │        (935 rows)
├── Client B/                    │  Parse fields     │
│   └── ...                      │  Validate & QA    │
└── Client Z/                    │  Deduplicate      │
    └── ...                      └─────────────────┘
                                         │
                                         │ missing email/phone/mobile
                                         ▼
                                    Email Enrichment
                                    ────────────────
Inbox/ + Sent/                   ┌─────────────────┐
├── 46,000 emails        ──────>│  Search archive   │──────> client_contacts_enriched.csv
│   (plain text .txt)            │  Match by name    │        (593/935 enriched)
                                 │  Extract from     │
                                 │  signatures       │
                                 └─────────────────┘
```

## Key Challenges

**Data quality:**
- One client appearing under 30 slightly different name variations
- Quote formats changed every 3-4 years; address blocks moved locations
- Mixed documents in client folders (supplier invoices, airline receipts, customs forms)
- Scanned image PDFs with no extractable text
- Unicode encoding issues on macOS (NFD vs NFC normalization)

**Document formats (5 distinct layouts):**
- French offers (2015-present): address block top-left
- English quotations newer (2020+): sender line above client address
- English quotations older (2013-2019): different header with old company address
- Invoices: sometimes missing address block entirely
- Proforma invoices: address appears after pricing table

**Email parsing:**
- Headers wrapping across multiple lines without consistent indentation
- Outlook forwarded emails using bold-formatted headers
- Quoted names in "Last, First" format

## What I'd Do Differently

**1. Classify documents before extracting.** Half the QA rounds were removing rows from third-party documents. An LLM classification pass upfront (~$15 for 5,000 PDFs with Haiku) would have eliminated most of these.

**2. LLM extraction instead of regex.** The regex address parser grew to handle ~15 layouts with hundreds of edge cases. A single LLM call per PDF would have handled 95% correctly on the first try.

**3. Calibration sample first.** Extract 5-10 PDFs per folder as a test set, build all rules from that, then one full run. Would have compressed 9 rounds into 2-3.

**4. Grep-first for email enrichment.** The indexed pipeline had bugs because it had an index. The grep approach has no index, so it has no index bugs. ~250 lines vs ~800 lines, equivalent results.

**5. Per-field confidence from the start.** A single confidence score per row meant high-confidence email + medium-confidence phone = medium overall, blocking the email from being applied.

See [LEARNINGS.md](LEARNINGS.md) for the full write-up on lessons learned.

## Project Structure

```
├── README.md
├── LEARNINGS.md                   # Personal reflections and lessons learned
├── docs/
│   ├── methodology.md             # Detailed extraction logic and heuristics
│   └── document-formats.md        # PDF layout descriptions and parsing strategies
├── examples/
│   └── sample_output.csv          # Anonymized example output
└── .gitignore
```

> **Note:** Source data (PDFs, emails, client CSVs) is not included for privacy reasons. Scripts reference specific folder structures and document patterns from the original project.

## License

MIT


