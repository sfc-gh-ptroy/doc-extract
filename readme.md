# PDF Extraction Demo — Overview

## What This Notebook Does

Extracts structured, queryable data from unstructured PDF documents using Snowflake's `AI_EXTRACT` function with typed `responseFormat` schemas.

Two document types are processed:

1. **Bank Statements** — Extracts transaction-level data (date, description, amount, balance) from 5 different bank/credit card statement PDFs
2. **Construction Spec Sheets** — Extracts equipment schedules (tags, service areas, CFM, ESP, materials, controllers) from 5 HVAC engineering specification PDFs

## Key Technique: responseFormat Schema

Instead of free-text prompts or `AI_PARSE_DOCUMENT` + string parsing, this notebook uses a **typed JSON schema** as the second argument to `AI_EXTRACT`:

```sql
AI_EXTRACT(
    file => TO_FILE('@my_stage', 'document.pdf'),
    responseFormat => {
        'schema': {
            'type': 'object',
            'properties': {
                'transactions': {
                    'type': 'object',
                    'column_ordering': ['date', 'description', 'amount', 'balance'],
                    'properties': {
                        'date':        { 'type': 'array', 'description': '...' },
                        'description': { 'type': 'array', 'description': '...' },
                        'amount':      { 'type': 'array', 'description': '...' },
                        'balance':     { 'type': 'array', 'description': '...' }
                    }
                }
            }
        }
    }
)
```

This returns **parallel arrays** — one array per column — which `LATERAL FLATTEN` unpacks into rows.

### Why This Approach

| Alternative | Problem |
|-------------|---------|
| Free-text prompt (`['transactions: array of objects...']`) | Output token limit truncates long documents (missed 50%+ of transactions) |
| `AI_PARSE_DOCUMENT` + `SPLIT_TO_TABLE` | Complex SQL, brittle to format changes, requires regex filtering |
| `responseFormat` schema (this notebook) | Full extraction, clean output, simple FLATTEN |

## Notebook Structure

| Section | What It Does |
|---------|--------------|
| Setup | Creates database, warehouse, and two internal stages |
| Upload | Instructions for uploading PDFs to stages |
| Verify | Lists files on stage to confirm upload |
| Bank Statements — Extract | Runs `AI_EXTRACT` with schema on all bank PDFs |
| Bank Statements — Flatten | LATERAL FLATTEN + edge case normalization (dollar signs, dashes, None values) |
| Construction Specs — Extract | Runs `AI_EXTRACT` with unified multi-doc-type schema |
| Construction Specs — Flatten | LATERAL FLATTEN with NULLIF for non-applicable columns |
| Cleanup | Drops all created resources |

## Edge Cases Handled

### Bank Statements
- `$` and `,` in amounts/balances → stripped with `REPLACE`
- `+` prefix on credits → stripped
- `—` (em-dash) on Balance Forward rows → stripped, returns NULL
- `None` values (credit cards with no running balance) → filtered out
- Inconsistent date formats across banks (`04/01` vs `Apr 01`)

### Construction Specs
- Multiple document types with different column sets (AHU, ductwork, controls, chiller, exhaust)
- Non-applicable fields → `NULLIF(..., 'None')` converts to proper SQL NULL
- Numeric values with commas and `%` signs → `TRY_TO_DOUBLE` + `REPLACE`

## Prerequisites

- Snowflake account with `AI_EXTRACT` access (Cortex AI functions enabled)
- PDF files uploaded to the respective stages
- `COMPUTE_WH` warehouse (XSMALL is sufficient)

## Output

The final queries produce one row per:
- **Transaction** (bank statements): file, document type, issuer, recipient, date, description, amount, balance
- **Equipment item** (construction specs): file, document number, section, tag, service, CFM, ESP, cooling, heating, OA%, material, gauge, etc.

These can be persisted to tables via `CREATE TABLE ... AS SELECT` or `INSERT INTO` for downstream analytics, semantic views, or agent consumption.
