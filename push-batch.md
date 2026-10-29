---
title: Batch Push API
description: Sending a complete invoice snapshot to Sunbay in a single request
permalink: /push-batch/
---

This page specifies the batch push endpoint as it is implemented: one HTTPS request carries a complete snapshot of the documents in scope. For integrations running on this endpoint it replaces the push proposal in §5 of the [integration guide](https://integration-docs.sunbay.io/).

**Last change: 2026-10-29** - `correctedInvoiceId` replaced by `correctedInvoiceIds` (see [Changelog](#changelog)).

## Contents

1. [Request](#1-request)
2. [Snapshot](#2-snapshot)
3. [Document fields](#3-document-fields)
4. [Corrections](#4-corrections)
5. [Payments and cancellations](#5-payments-and-cancellations)
6. [Responses](#6-responses)
7. [Example](#7-example)
8. [Changelog](#changelog)

---

## 1. Request

```
POST {baseUrl}/api/ingest/invoices
X-Api-Key: {apiKey}
Content-Type: application/json
```

- `baseUrl` and `apiKey` are provided per environment (test, production) at onboarding. The key is shown once; Sunbay can rotate or revoke it on request.
- Each header goes on its own line as `Name: value`, with no line breaks inside the value. A malformed header is rejected by the gateway before it reaches Sunbay, with HTTP `400` and an empty body.
- Request body: at most 30,000,000 bytes.
- Rate limit: 10 requests per 60 seconds.

## 2. Snapshot

```json
{
  "syncMode": "Snapshot",
  "generatedAt": "2026-10-29T08:00:00Z",
  "totalCount": 4,
  "invoices": [ ...documents, §3... ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `syncMode` | string | **Yes** | Always `Snapshot`. |
| `generatedAt` | timestamp | **Yes** | When the snapshot was produced. ISO 8601 UTC ending in `Z`. |
| `totalCount` | integer | **Yes** | Number of elements in `invoices`. Must match exactly. |
| `invoices` | array | **Yes** | The documents (§3). An empty array is a valid snapshot with nothing in scope. |

**Every request is a complete picture.** The accepted snapshot with the latest `generatedAt` replaces the previous one as a whole, so a document missing from it is no longer in scope. A snapshot contains:

1. every document with an open amount (`Open`, `PartiallyPaid`), whatever its age;
2. documents paid or cancelled within the agreed window (14 days);
3. every invoice listed in `correctedInvoiceIds` of a document in the snapshot, whatever its age or status (§4).

**Validation is all or nothing.** If any document fails, the whole request is rejected with `422`, nothing is stored and the previous snapshot stays in place. Sending the same snapshot again is safe.

## 3. Document fields

All fields are top-level properties of the document. There are no nested `customer` or `seller` objects.

**Identification and dates**

| Field | Type | Required | Description |
|---|---|---|---|
| `invoiceId` | string | **Yes** | Stable identifier of the document in the source system. The same document always carries the same id. Unique within the snapshot. |
| `invoiceNumber` | string | **Yes** | Number shown to the customer. Not required to be unique. |
| `documentType` | string | **Yes** | `Invoice` · `CorrectiveInvoice` · `AdvanceInvoice` · `FinalInvoice` · `Proforma` · `CreditNote` · `DebitNote`. Other values are accepted with a warning (§6). |
| `correctedInvoiceIds` | string[] | **Cond.** | Invoices a correction relates to (§4). |
| `issueDate` | date | **Yes** | Issue date. |
| `dueDate` | date | **Yes** | Due date. Required on every document. |
| `lastModifiedAt` | timestamp | **Yes** | When the document last changed. ISO 8601 UTC ending in `Z`. |

**Amounts**

| Field | Type | Required | Description |
|---|---|---|---|
| `currency` | string | **Yes** | ISO 4217, e.g. `PLN`, `EUR`. |
| `amountNet` | number | **Yes** | Net amount. Signed on corrections (§4). |
| `amountVat` | number | **Yes** | VAT amount. Signed on corrections. |
| `amountGross` | number | **Yes** | Gross amount. Signed on corrections. |
| `amountPaid` | number | **Yes** | Amount settled by payments, `0` if none. Never capped (§5). |
| `amountOutstanding` | number | **Yes** | Open amount of this document in the source system. |

`amountPaid + amountOutstanding` must equal `amountGross` exactly, on every document except `Cancelled` ones (§5).

**Status**

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | string | **Yes** | `Open` · `PartiallyPaid` · `Paid` · `Cancelled`. Other values are accepted with a warning (§6). |
| `paidDate` | date | **Opt.** | Date of full settlement. `null` while an amount is open, and on cancelled documents. |
| `isBlockedForCollection` | boolean | **Opt.** | `true` when the document must not be chased (dispute, legal hold). |

**Customer**

| Field | Type | Required | Description |
|---|---|---|---|
| `customerId` | string | **Yes** | Stable identifier of the customer in the source system. |
| `customerName` | string | **Yes** | Company or person name. |
| `customerCountryCode` | string | **Yes** | ISO 3166-1 alpha-2, e.g. `PL`. |
| `customerTaxId` | string | **Opt.** | Tax identifier, one consistent format across the feed. |
| `customerEmail` | string | **Opt.** | Main email address. Needed for email reminders. |
| `customerEmailCcs` | string[] | **Opt.** | Additional addresses in copy. |
| `customerPhone` | string | **Opt.** | International format with country code, e.g. `+48512345678`. Needed for SMS reminders. |
| `customerStreet` | string | **Opt.** | Street and number. |
| `customerPostalCode` | string | **Opt.** | Postal code. |
| `customerCity` | string | **Opt.** | City. |
| `customerCommunicationLanguage` | string | **Opt.** | ISO 639-1, e.g. `pl`, `en`. |
| `customerCustomFields` | object | **Opt.** | Customer-level key-value pairs. |

**Other**

| Field | Type | Required | Description |
|---|---|---|---|
| `bankAccount` | string | **Opt.** | Account the customer should pay into. |
| `customFields` | object | **Opt.** | Document-level key-value pairs. |

**Formats**

- Required text: non-empty, not whitespace only.
- Optional text: omit it or send `null` when unknown. An empty string `""` is rejected.
- Amounts: JSON numbers with a dot as the decimal separator, never strings.
- Dates: `YYYY-MM-DD`. Timestamps: ISO 8601 UTC ending in `Z`.
- Encoding: UTF-8.
- Fields not listed on this page are not part of the contract. Send additional attributes in `customFields` or `customerCustomFields`.

## 4. Corrections

- A correction is a **separate document** with its own `invoiceId` and `documentType` `CorrectiveInvoice`, `CreditNote` or `DebitNote`.
- Its amounts are the **signed difference** it introduces: negative lowers what the customer owes, positive raises it.
- The corrected invoice is **not adjusted** in the feed: it keeps its own amounts and `amountOutstanding` as in the source system.
- The correction is **one record with its full amount**, never split across the invoices it relates to.
- `correctedInvoiceIds` lists the invoices it relates to:

| Case | `correctedInvoiceIds` |
|---|---|
| Correction of one invoice | `["<invoiceId>"]` |
| Correction of several invoices | every `invoiceId` it relates to |
| Correction not related to a specific invoice, e.g. a retrospective bonus | `[]` |
| Document that is not a correction | omitted, `null` or `[]` |

- Each element is a non-empty string; no duplicates; never the document's own `invoiceId`. A violation rejects the request with `422`.
- Every listed invoice must be in the same snapshot (§2, point 3). A missing one produces a warning (§6); the snapshot is still accepted.

## 5. Payments and cancellations

| Case | Shape |
|---|---|
| Open | `amountPaid = 0`, `amountOutstanding = amountGross`, `status = Open` |
| Partially paid | `amountPaid` as booked, `amountOutstanding` the rest, `status = PartiallyPaid` |
| Paid | `amountOutstanding = 0`, `status = Paid`, `paidDate` set |
| Overpaid | `amountPaid` as booked, above `amountGross`; `amountOutstanding` negative by the surplus. Never capped |
| Cancelled | `status = Cancelled`, `amountGross` nominal, `amountPaid = 0`, `amountOutstanding = 0`, `paidDate = null`. The balance rule of §3 does not apply |

## 6. Responses

| Status | Meaning |
|---|---|
| `202 Accepted` | Snapshot accepted. Body below. |
| `400` with an empty body | Malformed HTTP header, rejected by the gateway (§1). |
| `401 Unauthorized` | Missing, wrong or revoked `X-Api-Key`. |
| `404 Not Found` | The feed is disabled on Sunbay's side. Contact us. |
| `422 Unprocessable Entity` | Validation failed; nothing was stored. Problem details (RFC 7807) with an `errors` object keyed by field path, e.g. `Payload.invoices[12].amountOutstanding`. |
| `429 Too Many Requests` | Rate limit exceeded (§1). Wait and retry. |
| `5xx` | Server error. Retry with backoff; resending the snapshot is safe. |

```json
{
  "batchId": "3f2b8c1e-7d4a-4e9b-9c61-0a5d2e8f4b17",
  "acceptedCount": 4,
  "warnings": []
}
```

`warnings` do not block the snapshot. Possible values:

- `Unknown documentType values: <values>.`
- `Unknown status values: <values>.`
- `Invoice '<invoiceId>' references correctedInvoiceIds missing from the batch: <ids>.`

## 7. Example

Two invoices, a correction relating to both, and a bonus not related to a specific invoice.

```json
{
  "syncMode": "Snapshot",
  "generatedAt": "2026-10-29T08:00:00Z",
  "totalCount": 4,
  "invoices": [
    {
      "invoiceId": "1000-2026-0000100001-001",
      "invoiceNumber": "900100001",
      "documentType": "Invoice",
      "issueDate": "2026-09-15",
      "dueDate": "2026-10-15",
      "lastModifiedAt": "2026-09-15T10:12:00Z",
      "currency": "PLN",
      "amountNet": 1000.00,
      "amountVat": 230.00,
      "amountGross": 1230.00,
      "amountPaid": 0.00,
      "amountOutstanding": 1230.00,
      "status": "Open",
      "paidDate": null,
      "isBlockedForCollection": false,
      "customerId": "0001000123",
      "customerName": "Example Bikes Sp. z o.o.",
      "customerCountryCode": "PL",
      "customerTaxId": "PL5213001234",
      "customerEmail": "accounts@example-bikes.pl",
      "customerPhone": "+48512345678",
      "customerStreet": "ul. Przykładowa 12",
      "customerPostalCode": "00-001",
      "customerCity": "Warszawa",
      "customerCommunicationLanguage": "pl",
      "bankAccount": "PL61109010140000071219812874",
      "customFields": { "companyCode": "1000" },
      "customerCustomFields": { "salesOrg": "1000" }
    },
    {
      "invoiceId": "1000-2026-0000100002-001",
      "invoiceNumber": "900100002",
      "documentType": "Invoice",
      "issueDate": "2026-08-20",
      "dueDate": "2026-09-19",
      "lastModifiedAt": "2026-09-18T14:30:00Z",
      "currency": "PLN",
      "amountNet": 2000.00,
      "amountVat": 460.00,
      "amountGross": 2460.00,
      "amountPaid": 2460.00,
      "amountOutstanding": 0.00,
      "status": "Paid",
      "paidDate": "2026-09-18",
      "customerId": "0001000123",
      "customerName": "Example Bikes Sp. z o.o.",
      "customerCountryCode": "PL"
    },
    {
      "invoiceId": "1000-2026-1400000501-001",
      "invoiceNumber": "910000501",
      "documentType": "CorrectiveInvoice",
      "correctedInvoiceIds": ["1000-2026-0000100001-001", "1000-2026-0000100002-001"],
      "issueDate": "2026-09-30",
      "dueDate": "2026-09-30",
      "lastModifiedAt": "2026-09-30T09:00:00Z",
      "currency": "PLN",
      "amountNet": -121.95,
      "amountVat": -28.05,
      "amountGross": -150.00,
      "amountPaid": 0.00,
      "amountOutstanding": -150.00,
      "status": "Open",
      "customerId": "0001000123",
      "customerName": "Example Bikes Sp. z o.o.",
      "customerCountryCode": "PL"
    },
    {
      "invoiceId": "1000-2026-1400000502-001",
      "invoiceNumber": "910000502",
      "documentType": "CorrectiveInvoice",
      "correctedInvoiceIds": [],
      "issueDate": "2026-10-01",
      "dueDate": "2026-10-01",
      "lastModifiedAt": "2026-10-01T09:00:00Z",
      "currency": "PLN",
      "amountNet": -406.50,
      "amountVat": -93.50,
      "amountGross": -500.00,
      "amountPaid": 0.00,
      "amountOutstanding": -500.00,
      "status": "Open",
      "customerId": "0001000123",
      "customerName": "Example Bikes Sp. z o.o.",
      "customerCountryCode": "PL"
    }
  ]
}
```

## Changelog

| Date | Change |
|---|---|
| 2026-10-29 | `correctedInvoiceId` (one string) replaced by `correctedInvoiceIds` (array). A correction lists every invoice it relates to, or `[]` when it relates to none. The old field is no longer part of the contract. |
