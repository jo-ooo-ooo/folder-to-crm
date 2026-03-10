# Document Formats

The source PDFs came in several layouts across different eras. Understanding these formats was essential for building the extraction pipeline.

## French Offers / Devis (2015–present)

```
┌─────────────────────────────────────────┐
│ Client Company Name                      │
│ Contact Person                           │
│ Street Address                           │
│ Postal Code + City                       │
│                                          │
│ Devis: REF-XXXXXX                        │
│ Production: [description]                │
│                                          │
│ [pricing table]                          │
│                                          │
│ Cordialement,                            │
│ [Employee Name] ← NOT the client         │
│                                          │
│ ─────────────────────────────────────    │
│ [Company footer: address, tax ID, bank]  │
│ ← ignore this block                      │
└─────────────────────────────────────────┘
```

**Key patterns:**
- Client address block is at the top-left
- The greeting + employee name at bottom is NOT the client contact
- Footer contains company info (must be filtered as boilerplate)
- Some address blocks include branch office lines between company and person — these are NOT person names

## English Quotations — Newer Format (2020+)

```
┌─────────────────────────────────────────┐
│ Company - Street 123 - D-XXXXX City     │  ← sender line
│                                          │
│ Client Company                           │
│ attn.: Contact Person                    │
│ Street Address                           │
│ Postal Code City                         │
│ Country                    ┌───────────┐ │
│                            │ Quotation │ │
│                            │ Date: ... │ │
│                            │ Ref: ...  │ │
│                            └───────────┘ │
│ [pricing table]                          │
└─────────────────────────────────────────┘
```

**Key patterns:**
- Sender line above client address (used for fallback strategy 2)
- Client address may include "attn.:" prefix for contact person
- Metadata box on right side with date, version, reference
- In some 2024+ documents, the client address appears AFTER the pricing table

## English Quotations — Older Format (2013–2019)

```
┌─────────────────────────────────────────┐
│ COMPANY | Old Street 36 | XXXXX City    │  ← old address header
│                                          │
│ Client Company                           │
│ Contact Person                           │
│ Street Address                           │
│ Postal Code City                         │
│                                          │
│ Purchase offer: [description]            │
│ [pricing table]                          │
└─────────────────────────────────────────┘
```

**Key patterns:**
- Different header format with pipe separators
- Old company address (different from current)
- "Purchase offer:" label instead of "Quotation"

## Invoices / Factures

```
┌─────────────────────────────────────────┐
│ [Similar header to offers]               │
│                                          │
│ Client Company                           │
│ Contact Person                           │
│ Address                                  │
│                                          │
│ Facture: FN-XXXXXX                       │
│ [line items]                             │
└─────────────────────────────────────────┘
```

**Key patterns:**
- Similar layout to offers but with invoice prefix (FN/RN)
- Some internally generated invoices have minimal address blocks
- Address block sometimes missing from extracted text entirely

## Proforma Invoices

```
┌─────────────────────────────────────────┐
│ Company - Street 123 - D-XXXXX City     │  ← sender line
│                                          │
│ [pricing table]                          │
│                                          │
│ Client Company                           │  ← address AFTER table
│ Contact Person                           │
│ Street Address                           │
│ Country                                  │
└─────────────────────────────────────────┘
```

**Key patterns:**
- PI prefix document references (e.g., PI20251020) — these are document IDs, not address parts
- Address block appears after sender line AND after pricing table
- Requires fallback extraction strategies

## Common Pitfalls

| Pattern | Looks like | Actually is |
|---------|-----------|-------------|
| Branch office line ("Succursale Lyon") | Person name | Location descriptor |
| City name ("Lagos", "Tripoli") | Person name | City |
| Employee name after greeting | Client contact | Internal employee |
| Footer company info | Client address | Sender's own details |
| Document reference (PI20251020) | Part of address | Document ID |
| "attn.: Person, Company suffix" | Single company line | Company + person on one line |
| Freelancer with no company | Missing company | Person name = company name |

## Non-Client Documents Found in Client Folders

These document types were mixed into client folders and had to be rejected:

- **Logistics:** Shipping receipts, import duty bills, freight charges, customs forms
- **Travel:** Airline boarding passes, hotel booking receipts, ride-hailing receipts
- **Financial:** Third-party invoices TO the company, freelancer invoices, tax declarations
- **Administrative:** Partnership declarations, vendor registration forms, warranty documents
- **Communications:** Forwarded email PDFs, offer requests FROM clients
- **Equipment:** Third-party rental invoices, packing lists, safety documentation
