# Methodology

## Overview

Two-phase pipeline: extract contacts from PDFs, then enrich missing fields from email archives.

## Phase 1: PDF Extraction

### Pipeline steps

1. **Find PDFs** — Recursively scan each client subfolder. Skip excluded paths (expense receipts, customs docs, third-party rentals, personnel files).
2. **Filter by path** — Skip files matching exclusion patterns. Uses NFC-normalized Unicode for macOS compatibility.
3. **Extract text** — PyMuPDF extracts text from first page (+ up to 3 pages for context).
4. **Classify document** — Only process if it's an outgoing commercial document (must reference company products AND contain a document type indicator like "Devis", "Facture", "Quotation"). Rejects third-party documents.
5. **Extract address block** — Primary: collect non-boilerplate lines from top of first page. Three fallback strategies if primary yields insufficient lines.
6. **Parse address block** — Classify each line as company, person, street, postal code, city, country, or unknown.
7. **Search company in text** — If no company extracted from address block, scan full first page for company suffix lines (SARL, Ltd, Inc, etc.).
8. **Extract phone/mobile/email** — Search header area for labeled contact fields.
9. **Cleanup** — Normalize phone numbers, remove company's own numbers, strip dates and document references from wrong fields, filter placeholder entries.
10. **QA validation** — Post-extraction sanity checks: reject bad company patterns (closing phrases, metadata labels), reject bad person patterns (job titles, labels).
11. **Deduplicate** — Using NFC-normalized, whitespace-collapsed keys.
12. **Write CSV** — With folder name as first column.

### Document classification

Documents are classified as outgoing commercial if they:
- Reference company products (equipment model names, system names)
- Contain a document type indicator (quote/offer/invoice reference numbers)
- Are NOT third-party documents

~50 rejection patterns filter out:
- Logistics receipts (shipping companies, airline bookings)
- Incoming purchase orders (identified by "Bill to:" containing company name)
- Tax forms, customs declarations
- Forwarded email PDFs
- Freelancer invoices addressed TO the company
- Equipment rental agreements from other vendors

### Address block extraction strategies

**Primary:** Collect lines from top of first page, skipping boilerplate (company's own name/address, employee names, footer text, garbled encoding).

**Fallback 1:** Search for attention patterns (`à l'att. de`, `attn`) anywhere on first page. Pre-filters phone/bank/price lines from context.

**Fallback 2:** Find the company sender line, collect client info from lines after it. Also searches for company names and person names before the sender line. Filters metadata labels.

**Fallback 3:** Search for company suffix lines (SARL, Ltd, Inc, GmbH) with surrounding context.

### Field classification heuristics

Each line is classified by pattern matching:

- **Company:** Contains corporate suffixes (SARL, Ltd, Inc, GmbH, LLC, etc.) or known indicators (Films, Holding, Hotel, University, Sports, Touring)
- **Person:** Validated with name particles (de, van, von, do, del, le, la) and uppercase word checks. Blocked by non-person first words (branch office, department, headquarters terms)
- **Street:** Strong indicators (str., rue, avenue, way + digit, quarter + digit)
- **Postal code:** Digit patterns with optional country prefixes (F-75001, SE-164 46)
- **City:** Checked against known cities list (~35 entries) to prevent misclassification as person names
- **Country:** Comprehensive list (~272 entries) from pycountry with translations and common misspellings

Post-classification fixups:
- Company/street overlap: strong street indicators override weak company matches
- Person before country: single-word "person" reclassified as city when followed by country
- Freelancer rule: if no company but person exists, company = person
- Company/person swap: cross-validated against folder name

### Incoming document detection

Multi-line company address detection — if the company name appears on its own line with the street address on a separate nearby line, it indicates the company is the addressee (incoming doc). Also detects "TO: [company]" patterns and employee names in addressee context.

## Phase 2: Email Enrichment

### Approach 1: Indexed pipeline (~800 lines)

**Indexing:**
- Parse all .txt email files, skip logistics/spam
- Filter out company's own domains
- Build three indexes: domain → files, person name → files, PDF filename → files

**Matching (4 strategies, priority order):**
1. PDF attachment name — exact match of source file basename
2. Person name — first + last name words in same index key
3. Company domain — company name words matched against email domains
4. Folder name — folder name words matched against domains and subjects

**Extraction:** Send matched email excerpts to Claude Sonnet for structured extraction. Cache results for resume support.

### Approach 2: Regex extraction (no API)

Pure regex extraction from pre-matched candidate emails. Extracts email addresses, phone numbers (Tel/Phone/Fon labels), and mobile numbers (Mob/Cell/Handy labels) from signature blocks. High-confidence results applied directly, medium-confidence queued for review.

### Approach 3: Grep-based (~250 lines)

For each row missing fields:
1. Search all email files for person name, company name, or PDF filename
2. Score matches by match type (person > PDF > company > folder)
3. Extract email from headers near person name (`Person <email>` patterns)
4. Extract phone/mobile from signature blocks of emails sent by the person

### Enrichment strategies

- **Email prefix matching:** Generate firstname.lastname patterns, match against archive headers
- **Known-domain propagation:** Use domains from enriched rows to find other contacts at same company
- **Cross-row propagation:** Copy contact fields between rows with same person name
- **Sender signature scan:** Find phones in signatures from people who sent emails to the company

## Quality Assurance

- Manual sampling after each extraction round
- QA bad patterns: reject companies/persons that are actually metadata labels, product descriptions, financial text, page numbers, greeting phrases
- Phone normalization: all values to +XXXXXXXXXXX format, trunk-0 stripped, country codes inferred from address context
- Deduplication: NFC-normalized, whitespace-collapsed keys
- Cross-validation: company/person fields checked against folder names
