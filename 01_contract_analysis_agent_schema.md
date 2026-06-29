# Contract Commercial Terms Analyst — Agent Schema Reference
> Version: 1.0 | Last updated: 2026-06-21
> Status: BUILT AND TESTED in UiPath Studio Web sandbox. Confirmed working
> against a real sample contract (Brightline Solutions / Northwind Retail).

---

## Overview

This agent accepts a contract document (PDF, DOCX, etc.), extracts all key
commercial terms, and returns a structured JSON output with per-field confidence
scores, an overall confidence roll-up, and a human-review flag.

**Tool used:** Analyze Files (built-in)
**Model:** gpt-5.4 (suggested default in this Studio Web tenant)
**Agent type:** Autonomous Agent, Standard harness

---

## Input Schema

| Field            | Type            | Required | Description                                                  |
|------------------|-----------------|----------|--------------------------------------------------------------|
| contractDocument | file attachment | Yes      | The contract file (PDF, DOCX, or similar) to be analyzed.    |

### Raw JSON

```json
{
  "type": "object",
  "required": ["contractDocument"],
  "properties": {
    "contractDocument": {
      "$ref": "#/definitions/job-attachment",
      "description": "The contract file (PDF, DOCX, or similar) to be analyzed for key commercial terms. Must be a valid file attachment uploaded by the user."
    }
  },
  "definitions": {
    "job-attachment": {
      "type": "object",
      "required": ["ID"],
      "x-uipath-resource-kind": "JobAttachment",
      "properties": {
        "ID":       { "type": "string", "description": "Orchestrator attachment key" },
        "FullName": { "type": "string", "description": "File name" },
        "MimeType": { "type": "string", "description": "MIME type, e.g. \"application/pdf\"" },
        "Metadata": {
          "type": "object",
          "description": "Dictionary<string, string> of metadata",
          "additionalProperties": { "type": "string" }
        }
      }
    }
  }
}
```

---

## Output Schema

All 13 top-level fields are **required** in every response.

### Field Reference

| # | Field               | Type    | Description                                                                                          |
|---|---------------------|---------|------------------------------------------------------------------------------------------------------|
| 1 | `contractReferenceId` | string | Contract reference number/ID as printed in the document. Empty string if not found.                  |
| 2 | `contractStartDate`   | string | Effective/start date in ISO 8601 (YYYY-MM-DD) where possible. Empty string if not found.             |
| 3 | `buyerName`           | string | Full legal name of the buying/client party. Empty string if not found.                               |
| 4 | `sellerName`          | string | Full legal name of the selling/vendor party. Empty string if not found.                              |
| 5 | `contractedPrice`     | object | See sub-fields below.                                                                                |
|   | -> `amount`            | string | Numeric price value as a string (e.g. "100000"). Empty string if not found.                         |
|   | -> `currency`          | string | Currency code or symbol (e.g. "INR", "USD"). Empty string if not found.                             |
|   | -> `period`            | string | Period the price covers (e.g. "monthly", "annually"). Empty string if not found.                    |
| 6 | `billingFrequency`    | string | How often the client is billed (e.g. "monthly", "quarterly"). Empty string if not stated.           |
| 7 | `priceEscalation`     | object | See sub-fields below.                                                                                |
|   | -> `hasEscalation`     | boolean | true if an explicit escalation clause exists; false otherwise.                                      |
|   | -> `rate`              | string | Escalation rate or formula (e.g. "10% per annum"). Empty string if not applicable.                  |
|   | -> `appliesFrom`       | string | When escalation first applies (e.g. "2026-01-01"). Empty string if not specified.                   |
|   | -> `conditions`        | string | Any conditions or triggers for the escalation clause. Empty string if none.                         |
| 8 | `renewalTerms`        | object | See sub-fields below.                                                                                |
|   | -> `autoRenewal`       | boolean | true if contract auto-renews; false if manual or not stated.                                        |
|   | -> `renewalDate`       | string | Renewal/expiry date in ISO 8601 or descriptive string. Empty string if not found.                   |
|   | -> `noticePeriod`      | string | Notice period to cancel or prevent renewal (e.g. "60 days"). Empty string if not stated.            |
|   | -> `termLength`        | string | Length of each contract term (e.g. "12 months"). Empty string if not stated.                        |
| 9 | `discounts`           | array  | All discount clauses found. Empty array `[]` if none. See item sub-fields below.                    |
|   | -> `description`       | string | Short label (e.g. "Year 1 introductory discount").                                                   |
|   | -> `value`             | string | Discount amount or percentage (e.g. "10%", "$200 off"). Empty string if not quantified.             |
|   | -> `expiryDate`        | string | When the discount expires (ISO 8601 or descriptive). Empty string if not specified.                  |
|   | -> `conditions`        | string | Conditions for the discount to apply. Empty string if unconditional.                                 |
|10 | `summary`             | string | 3-6 sentence plain-English summary of all key terms, including both party names.                    |
|11 | `confidenceScores`    | object | One `{ score, note }` entry per extractable field (9 entries). See Confidence Scoring below.         |
|12 | `overallConfidence`   | string | Lowest score across all confidenceScores entries. Enum: `"high"` / `"medium"` / `"low"`.            |
|13 | `flaggedForReview`    | boolean | true if overallConfidence is "medium" or "low". false only when every field scored "high".          |

### Raw JSON

```json
{
  "type": "object",
  "required": [
    "contractReferenceId", "contractStartDate", "buyerName", "sellerName",
    "contractedPrice", "billingFrequency", "priceEscalation", "renewalTerms",
    "discounts", "summary", "confidenceScores", "overallConfidence", "flaggedForReview"
  ],
  "properties": {
    "contractReferenceId": { "type": "string", "description": "The contract reference number or ID as stated in the document (e.g. 'BLS-2025-0142'). Use empty string if not found." },
    "contractStartDate": { "type": "string", "description": "The contract start or effective date in ISO 8601 format (YYYY-MM-DD) where possible. Use empty string if not found." },
    "buyerName": { "type": "string", "description": "The full legal name of the buyer (client/customer) party. Use empty string if not found." },
    "sellerName": { "type": "string", "description": "The full legal name of the seller (vendor/service provider) party. Use empty string if not found." },
    "contractedPrice": {
      "type": "object",
      "description": "The contracted price or rate found in the contract.",
      "properties": {
        "amount":   { "type": "string", "description": "Numeric price value as a string (e.g. '100000'). Empty string if not found." },
        "currency": { "type": "string", "description": "Currency code or symbol (e.g. 'INR', 'USD', 'GBP'). Empty string if not found." },
        "period":   { "type": "string", "description": "Period the price covers (e.g. 'monthly', 'annually'). Empty string if not found." }
      }
    },
    "billingFrequency": { "type": "string", "description": "How often the client is billed (e.g. 'monthly', 'quarterly', 'annually'). Use empty string if not stated." },
    "priceEscalation": {
      "type": "object",
      "description": "Details of any price escalation or increase clause in the contract.",
      "properties": {
        "hasEscalation": { "type": "boolean", "description": "True if an explicit escalation clause exists, false otherwise." },
        "rate":          { "type": "string",  "description": "Escalation rate or formula (e.g. '10% per annum'). Empty string if not applicable." },
        "appliesFrom":   { "type": "string",  "description": "When escalation first applies (e.g. 'Year 2', '2026-01-01'). Empty string if not specified." },
        "conditions":    { "type": "string",  "description": "Conditions or triggers for the escalation clause. Empty string if none." }
      }
    },
    "renewalTerms": {
      "type": "object",
      "description": "Contract renewal details extracted from the document.",
      "properties": {
        "autoRenewal":  { "type": "boolean", "description": "True if the contract auto-renews; false if manual or not stated." },
        "renewalDate":  { "type": "string",  "description": "Renewal/expiry date in ISO 8601 (YYYY-MM-DD) or descriptive string. Empty string if not found." },
        "noticePeriod": { "type": "string",  "description": "Notice period to cancel or not renew (e.g. '60 days'). Empty string if not stated." },
        "termLength":   { "type": "string",  "description": "Length of each contract term (e.g. '12 months', '3 years'). Empty string if not stated." }
      }
    },
    "discounts": {
      "type": "array",
      "description": "All discount clauses found in the contract. Empty array [] if none exist.",
      "items": {
        "type": "object",
        "properties": {
          "description": { "type": "string", "description": "Short label (e.g. 'Year 1 introductory discount')." },
          "value":       { "type": "string", "description": "Discount amount or percentage (e.g. '10%'). Empty string if not quantified." },
          "expiryDate":  { "type": "string", "description": "When the discount expires (ISO 8601 or descriptive). Empty string if not specified." },
          "conditions":  { "type": "string", "description": "Conditions for the discount to apply. Empty string if unconditional." }
        }
      }
    },
    "summary": { "type": "string", "description": "3-6 sentence plain-English summary of all key commercial terms, written for a non-legal reader. Must include both party names." },
    "confidenceScores": {
      "type": "object",
      "description": "Per-field confidence ratings — one { score, note } entry per extractable field.",
      "properties": {
        "contractReferenceId": { "type": "object", "properties": { "score": { "type": "string", "enum": ["high","medium","low"] }, "note": { "type": "string" } } },
        "contractStartDate":   { "type": "object", "properties": { "score": { "type": "string", "enum": ["high","medium","low"] }, "note": { "type": "string" } } },
        "buyerName":           { "type": "object", "properties": { "score": { "type": "string", "enum": ["high","medium","low"] }, "note": { "type": "string" } } },
        "sellerName":          { "type": "object", "properties": { "score": { "type": "string", "enum": ["high","medium","low"] }, "note": { "type": "string" } } },
        "contractedPrice":     { "type": "object", "properties": { "score": { "type": "string", "enum": ["high","medium","low"] }, "note": { "type": "string" } } },
        "billingFrequency":    { "type": "object", "properties": { "score": { "type": "string", "enum": ["high","medium","low"] }, "note": { "type": "string" } } },
        "priceEscalation":     { "type": "object", "properties": { "score": { "type": "string", "enum": ["high","medium","low"] }, "note": { "type": "string" } } },
        "renewalTerms":        { "type": "object", "properties": { "score": { "type": "string", "enum": ["high","medium","low"] }, "note": { "type": "string" } } },
        "discounts":           { "type": "object", "properties": { "score": { "type": "string", "enum": ["high","medium","low"] }, "note": { "type": "string" } } }
      }
    },
    "overallConfidence": { "type": "string", "enum": ["high", "medium", "low"], "description": "The lowest confidence score across all confidenceScores entries. 'low' if any field is low; 'medium' if any field is medium; 'high' only if all fields are high." },
    "flaggedForReview": { "type": "boolean", "description": "True if any confidenceScores entry is 'medium' or 'low'. False only when every field scored 'high'." }
  }
}
```

---

## Confidence Scoring Rules

| Score    | When to assign                                                                 |
|----------|--------------------------------------------------------------------------------|
| `high`   | Term is explicitly and unambiguously stated in the contract text.              |
| `medium` | Term is implied, inferred, or only partially stated.                           |
| `low`    | Term is absent from the contract or highly ambiguous.                          |

### Special Rule — `billingFrequency`
Assign **`high`** if ANY of the following are found (in priority order):
1. An explicit label: "billed monthly", "billing frequency: annual", etc.
2. An invoice issuance cadence statement: "invoices shall be issued monthly", etc. (treated as a direct, reliable proxy).
3. A periodic payment schedule table with regular due dates.

Only assign `medium` if frequency must be indirectly inferred (e.g. from term length alone).
Only assign `low` if no billing-related language exists at all.

### Roll-up Rules
- `overallConfidence` = lowest individual score across all 9 fields.
- `flaggedForReview` = `true` if `overallConfidence` is `"medium"` or `"low"`.

---

## Empty / Missing Value Conventions

| Field type | Value when not found in contract |
|------------|-----------------------------------|
| `string`   | `""` (empty string)               |
| `boolean`  | `false`                           |
| `array`    | `[]` (empty array)                |

---

## CONFIRMED REAL TEST RESULT (Brightline Contract — BLS-2025-0142)

This is real, verified output from running the agent in UiPath Studio Web
against our own generated sample contract (Sample_Contract_Brightline_Solutions.pdf).
NOT hypothetical — this actually ran successfully.

```json
{
  "contractReferenceId": "BLS-2025-0142",
  "contractStartDate": "2025-01-01",
  "buyerName": "Northwind Retail Holdings Pvt. Ltd.",
  "sellerName": "Brightline Solutions Pvt. Ltd.",
  "contractedPrice": {
    "amount": "100000",
    "currency": "INR",
    "period": "monthly"
  },
  "billingFrequency": "monthly",
  "priceEscalation": {
    "hasEscalation": true,
    "rate": "10% per annum",
    "appliesFrom": "2026-01-01",
    "conditions": "Applies automatically on each anniversary of the Effective Date. Seller must provide at least 30 days written notice prior to each anniversary confirming the revised Monthly Fee."
  },
  "renewalTerms": {
    "autoRenewal": true,
    "renewalDate": "2026-01-01",
    "noticePeriod": "60 days prior to expiration of the then-current term",
    "termLength": "12 months"
  },
  "discounts": [
    {
      "description": "Year 1 first-year introductory discount",
      "value": "10% off Monthly Fee (effective fee: INR 90,000 per month)",
      "expiryDate": "2025-12-31",
      "conditions": "Applies during the Initial Term only (2025-01-01 to 2025-12-31). Does not apply to any Renewal Term or to the escalated fee."
    }
  ],
  "summary": "This agreement (ref: BLS-2025-0142) is between Northwind Retail Holdings Pvt. Ltd. (buyer) and Brightline Solutions Pvt. Ltd. (seller), effective January 1, 2025 for an initial 12-month term. The base monthly fee is INR 1,00,000, reduced to INR 90,000 during Year 1 only via a 10% introductory discount that expires December 31, 2025. From January 1, 2026, the fee escalates by 10% annually on each contract anniversary, with 30 days prior written notice required from the seller. The contract auto-renews for successive 12-month periods unless either party provides 60 days written notice before the end of the current term.",
  "confidenceScores": {
    "contractReferenceId": { "score": "high",   "note": "" },
    "contractStartDate":   { "score": "high",   "note": "" },
    "buyerName":           { "score": "high",   "note": "" },
    "sellerName":          { "score": "high",   "note": "" },
    "contractedPrice":     { "score": "high",   "note": "" },
    "billingFrequency":    { "score": "high",   "note": "" },
    "priceEscalation":     { "score": "high",   "note": "" },
    "renewalTerms":        { "score": "high",   "note": "" },
    "discounts":           { "score": "high",   "note": "" }
  },
  "overallConfidence": "high",
  "flaggedForReview": false
}
```

## NEXT AGENT DEPENDENCY NOTE

The Pricing Compliance Agent (Agent #2) will need to compare:
- `contractedPrice.amount` + `discounts[].value` + `discounts[].expiryDate`
  + `priceEscalation` against the actual invoice amount and invoice date,
  to determine if the currently-billed amount is correct for that point in time.
- Use `discounts[].expiryDate` to check if a discount that should have expired
  is still being applied (this is exactly our demo leakage scenario: discount
  expired 2025-12-31, but Feb 2026 invoice still shows the discounted rate).
