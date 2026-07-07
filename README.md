# ERP → Sunbay Invoice Data Integration Guide

This document specifies the **invoice data Sunbay needs to obtain** from your ERP / source system, and **the two ways that data can be exchanged**, so that Sunbay can run debt collection and receivables analytics on your behalf.

It is written for the **IT team building the integration** between the ERP landscape and Sunbay.

Two integration options are supported:

- **Option A (recommended)** - you host a small read-only API; **Sunbay pulls** invoices on its own schedule (§4).
- **Option B** - your system **pushes** invoices to Sunbay's ingestion endpoints (§5).

The **data model (§3) is the core of this document** and is identical for both options. Sections §4 and §5 describe the two exchange mechanisms; §6-§9 cover formats, security, reliability and scheduling. A number of protocol details are deliberately left **open for onboarding (§10)**.

> **Scope.** This document covers *what data Sunbay needs* and *how it is exchanged*. How Sunbay stores, processes, or acts on the data internally is intentionally out of scope.

---

## Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [Integration Options](#2-integration-options)
3. [Data Model](#3-data-model)
4. [Option A: Client-Hosted Invoice API (Sunbay Pulls)](#4-option-a-client-hosted-invoice-api-sunbay-pulls)
5. [Option B: Push to Sunbay](#5-option-b-push-to-sunbay)
6. [Data Formats & Conventions](#6-data-formats--conventions)
7. [Security & Authentication](#7-security--authentication)
8. [Reliability](#8-reliability)
9. [Scheduling & Volume](#9-scheduling--volume)
10. [Points to Confirm During Onboarding](#10-points-to-confirm-during-onboarding)

---

## 1. Purpose & Scope

Sunbay automates the collection of overdue receivables (email/SMS reminders and related flows) and provides receivables analytics. To do this, Sunbay needs a continuous feed of your **sales/revenue invoices - both paid and unpaid** - together with the **debtor (customer) data** for each invoice:

- **Unpaid and overdue invoices** drive the collection processes.
- **Paid invoices** power analytics: payment-behaviour insight, debtor scoring, aging and cash-flow reporting.

Optionally, the **PDF** of an invoice can be exchanged as well - it is needed **only** when invoice documents should be attached to reminder emails (§3.7).

In both integration options the ERP remains the **system of record**. Sunbay never writes back to the ERP - the integration is read-only from the ERP's perspective.

**In scope for this document**

- The exact data fields Sunbay needs per invoice and per customer (§3).
- The two exchange mechanisms: a client-hosted API that Sunbay polls (§4), or pushes to Sunbay's ingestion endpoints (§5).
- Data formats (§6), authentication (§7), reliability rules (§8), scheduling and volumes (§9).

**Out of scope**

- What Sunbay does with the data internally and how collection flows are configured.
- Storage of PDFs and any internal references/URLs Sunbay generates.

---

## 2. Integration Options

### 2.1 Option A - client-hosted API, Sunbay pulls (recommended)

You expose a small, **read-only HTTPS API** over the ERP data (specified in §4). Sunbay calls it on a schedule it controls: incremental polls for new and changed invoices, plus optional periodic full snapshots for reconciliation.

Why this is the recommended option:

- You implement **one read-only endpoint** (plus an optional PDF endpoint) - no scheduler, no retry logic, no outbound delivery pipeline on your side.
- Sunbay owns scheduling, backfill, retries and pacing - and can adapt them without any change on your side.
- Cancellations and corrections propagate naturally: Sunbay simply observes the current state of your data.
- The endpoint can be locked down tightly: Sunbay calls from a stable, published set of egress IP addresses which you can allowlist (§7.2).

### 2.2 Option B - push to Sunbay

Your system pushes each invoice to Sunbay's ingestion endpoints (§5), one call per invoice, on a schedule you run. Choose this option when connectivity is a hard constraint - i.e. no inbound endpoint may be exposed from your network, and your systems can only **send data out** over the public internet.

### 2.3 Comparison

| Aspect | Option A - Sunbay pulls | Option B - you push |
|---|---|---|
| Who initiates | Sunbay (scheduled polls) | Your system (own scheduler) |
| You implement | Read-only invoice endpoint (+ optional PDF endpoint) | A delivery job: per-invoice pushes, retries, backoff |
| Scheduling & retries | Owned by Sunbay | Owned by you |
| Cancellations | Visible as `Cancelled` tombstones in API responses (§4.5) | Require an explicit cancellation signal in incremental mode (§5.3) |
| Backfill / first load | Sunbay crawls the history via the same API | Bulk file upload or many individual pushes (§5.2) |
| PDFs (if enabled) | Fetched by Sunbay per invoice, once | Shipped by you together with each push |
| Tenant identification | Implicit - credentials and base URL are client-specific (§7.2) | `tenantCode` field in every payload (§7.3) |
| IP allowlisting | Feasible - Sunbay's egress IPs are stable | Generally impractical with dynamic outbound IPs |
| Result reporting | HTTP responses of your API | Acknowledgment design to be agreed (§5.5) |

### 2.4 Common to both options

- The **data model (§3)** and **formats (§6)** are identical.
- **Idempotency** is keyed on `externalInvoiceId` (§8) - re-delivery or re-fetch of the same invoice is a safe update, never a duplicate.
- Both **paid and unpaid** invoices are in scope.
- **PDFs are optional** in both options - they are needed only for email attachments (§3.7).

---

## 3. Data Model

This is the **most important section**. Each invoice exposed to (or pushed to) Sunbay should carry the following fields. Customer (debtor) data is **embedded with each invoice** (denormalized) - there is no separate customer feed.

**Required** column legend: **Yes** = mandatory · **Rec.** = recommended (strongly preferred) · **Opt.** = optional · **Cond.** = conditional.

### 3.1 Invoice identification

| Field | Type | Required | Description |
|---|---|---|---|
| `externalInvoiceId` | string | **Yes** | Stable, globally-unique identifier of the invoice **in the source system**. Used to recognise the same invoice across syncs (deduplication / update key). Must be **stable** - the same invoice must always carry the same id, even after edits. |
| `invoiceNumber` | string | **Yes** | Human-readable invoice number (e.g. `FV/2026/01/0123`). Shown to the debtor in reminders. |
| `documentType` | enum | **Yes** | Kind of document - see §3.1.1. |
| `correctedExternalInvoiceId` | string | **Cond.** | When `documentType = CorrectiveInvoice`: the `externalInvoiceId` of the original invoice this document corrects. |
| `issueDate` | date | **Yes** | Date the invoice was issued. |
| `dueDate` | date | **Yes** | Payment due date - when the invoice becomes collectible. |
| `paymentTermDays` | integer | **Opt.** | Payment term in days, if available. |

#### 3.1.1 `documentType` values

| Value | Meaning | Collectible receivable? |
|---|---|---|
| `Invoice` | Standard (VAT) invoice | **Yes** |
| `CorrectiveInvoice` | Corrective / adjustment invoice | Adjusts an existing receivable |
| `AdvanceInvoice` | Advance payment invoice | Yes (advance) |
| `FinalInvoice` | Final invoice | Yes |
| `Proforma` | Pro forma invoice | **No** - not a legal receivable |
| `CreditNote` | Credit note | Reduces a receivable |
| `DebitNote` | Debit note | Increases a receivable |

> If your source system uses other document kinds, list them during onboarding so we can map them (§10).

#### 3.1.2 Corrections

A corrective document carries `documentType = CorrectiveInvoice` and `correctedExternalInvoiceId` pointing at the original invoice. It adjusts the outstanding amount of the original receivable. **How your source system expresses correction amounts** (the new corrected totals, the difference/delta, or "before/after") must be confirmed during onboarding so the balance is interpreted correctly (§10).

### 3.2 Amounts & currency

| Field | Type | Required | Description |
|---|---|---|---|
| `currency` | string (ISO 4217) | **Yes** | e.g. `PLN`, `EUR`. |
| `amountNet` | decimal | **Yes** | Net amount. |
| `amountVat` | decimal | **Yes** | VAT amount. |
| `amountGross` | decimal | **Yes** | Gross total - the full amount of the receivable. |
| `amountPaid` | decimal | **Yes** | Amount already paid against this invoice (`0` if none). Enables partial-payment handling. |
| `amountOutstanding` | decimal | **Yes** | Remaining balance still owed (`amountGross - amountPaid`). This is what collection chases. |

### 3.3 Status & lifecycle

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | enum | **Yes** | `Open` · `PartiallyPaid` · `Paid` · `Cancelled`. |
| `paidDate` | date | **Cond.** | Date the invoice was fully paid. Required when `status = Paid`. |
| `isBlockedForCollection` | boolean | **Yes** | `true` if Sunbay must **not** chase this invoice (dispute, legal hold, internal block). |
| `lastModifiedAt` | timestamp | **Yes** (Option A) / **Rec.** (Option B) | When the invoice record last changed in the source system (ISO-8601 UTC). Drives incremental fetching in Option A (§4.2): **any** data change - status, amounts, payments, cancellation, correction linkage - must update this timestamp. |

### 3.4 Debtor (customer)

Embedded per invoice. This is the party Sunbay contacts.

| Field | Type | Required | Description |
|---|---|---|---|
| `externalCustomerId` | string | **Yes** | Stable, unique identifier of the customer in the source system. |
| `customerName` | string | **Yes** | Debtor name (company or person). |
| `customerTaxId` | string | **Rec.** | Tax identifier (e.g. VAT ID / NIP). |
| `customerEmail` | string | **Rec.** | Primary email. Required for email reminders. |
| `customerEmailCcs` | string[] | **Opt.** | **List** of additional CC email addresses. |
| `customerPhone` | string | **Opt.** | Phone number in **international format including the country code**, e.g. `+48512345678`. Required for SMS reminders. |
| `customerAddress` | string | **Rec.** | Postal address. |
| `customerCountryCode` | string (ISO 3166-1) | **Yes** | e.g. `PL`. |
| `customerCommunicationLanguage` | string (ISO 639-1) | **Opt.** | Preferred language for reminders, e.g. `pl`, `en`, `de`. Sunbay selects the reminder template language per debtor; when absent, the client-wide default is used. |
| `customFields` | object | **Opt.** | Arbitrary key-value pairs at **customer** level (see §3.6). |

### 3.5 Seller & payment

| Field | Type | Required | Description |
|---|---|---|---|
| `sellerName` | string | **Opt.** | Issuing entity name. A single installation may contain several legal entities/sellers. |
| `sellerTaxId` | string | **Opt.** | Seller tax identifier. |
| `bankAccount` | string | **Yes** | Bank account the debtor should pay into (IBAN/NRB). Included in reminders. |

### 3.6 Custom fields & references

| Field | Type | Required | Description |
|---|---|---|---|
| `customFields` | object | **Opt.** | Arbitrary key-value pairs at **invoice** level. **Custom fields are supported at both invoice and customer level** - send any extra attributes that may be useful (segment, region, contract code, cost centre, ...). |
| `externalReference` | string | **Opt.** | Any additional reference useful for reconciliation. |

### 3.7 PDF (optional)

> **PDFs are optional.** A PDF is needed **only if** you want Sunbay to attach the invoice document to reminder emails. If you do not need attachments, skip PDFs entirely - the integration is fully functional without them.

If PDF attachments are enabled:

- **Option A:** Sunbay fetches the PDF from your PDF endpoint (§4.3) - once per invoice on first ingestion, and again after a corrective document. You never ship PDFs proactively.
- **Option B:** the PDF is delivered together with the invoice data in the same push (§5.1).

One PDF per invoice. PDF specifics (always available? maximum size? PDFs for corrective documents?) are confirmed during onboarding (§10).

### 3.8 Line items (optional)

Invoice line items are **optional but valuable** - they enable richer analytics and more informative reminder content. The invoice is fully usable without them; when PDFs are exchanged, the PDF remains the authoritative document.

If provided, `lineItems` is an array where each line carries:

| Field | Type | Required (within a line) | Description |
|---|---|---|---|
| `name` | string | **Yes** | Description of the goods/service. |
| `quantity` | decimal | **Yes** | Quantity. |
| `unit` | string | **Opt.** | Unit of measure, e.g. `pcs`, `hours`. |
| `unitPriceNet` | decimal | **Yes** | Net unit price. |
| `vatRate` | decimal | **Rec.** | VAT rate in percent, e.g. `23.0`. |
| `vatAmount` | decimal | **Opt.** | VAT amount for the line. |
| `amountNet` | decimal | **Rec.** | Net total for the line. |
| `amountGross` | decimal | **Yes** | Gross total for the line. |

### 3.9 Example invoice object

The same invoice object is used in both options: it is the item shape returned by your API (Option A, §4.2) and the `invoice` part of a push payload (Option B, §5.1).

```json
{
  "externalInvoiceId": "ERP-2026-INV-000123",
  "invoiceNumber": "FV/2026/01/0123",
  "documentType": "Invoice",
  "correctedExternalInvoiceId": null,
  "issueDate": "2026-01-10",
  "dueDate": "2026-01-24",
  "paymentTermDays": 14,

  "currency": "PLN",
  "amountNet": 1000.00,
  "amountVat": 230.00,
  "amountGross": 1230.00,
  "amountPaid": 0.00,
  "amountOutstanding": 1230.00,

  "status": "Open",
  "paidDate": null,
  "isBlockedForCollection": false,
  "lastModifiedAt": "2026-01-25T11:02:14Z",

  "seller": {
    "sellerName": "ACME Sp. z o.o.",
    "sellerTaxId": "5213001234",
    "bankAccount": "PL61109010140000071219812874"
  },

  "customer": {
    "externalCustomerId": "ERP-CUST-10001",
    "customerName": "Kowalski Handel Sp. z o.o.",
    "customerTaxId": "7010001234",
    "customerEmail": "ksiegowosc@kowalski.pl",
    "customerEmailCcs": ["zarzad@kowalski.pl", "biuro@kowalski.pl"],
    "customerPhone": "+48512345678",
    "customerAddress": "ul. Przykładowa 12, 00-001 Warszawa",
    "customerCountryCode": "PL",
    "customerCommunicationLanguage": "pl",
    "customFields": {
      "segment": "B2B",
      "region": "Mazowieckie"
    }
  },

  "lineItems": [
    {
      "name": "Consulting services - January 2026",
      "quantity": 10.0,
      "unit": "hours",
      "unitPriceNet": 80.00,
      "vatRate": 23.0,
      "vatAmount": 184.00,
      "amountNet": 800.00,
      "amountGross": 984.00
    },
    {
      "name": "License fee",
      "quantity": 1.0,
      "unit": "pcs",
      "unitPriceNet": 200.00,
      "vatRate": 23.0,
      "vatAmount": 46.00,
      "amountNet": 200.00,
      "amountGross": 246.00
    }
  ],

  "externalReference": "SRC-REF-998877",
  "customFields": {
    "costCenter": "CC-204",
    "salesRep": "A. Nowak"
  }
}
```

---

## 4. Option A: Client-Hosted Invoice API (Sunbay Pulls)

This is the contract Sunbay's fetcher will code against. **Host and base path are yours to choose** - a versioned prefix is recommended, e.g. `{baseUrl} = https://api.example.com/sunbay/v1`. Everything below the base URL - paths, parameters, response shapes - should be implemented as specified here; deviations can be discussed during onboarding.

### 4.1 Overview & responsibilities

- You host a **read-only** HTTPS API exposing invoices in the shape defined in §3. Sunbay polls it on an agreed schedule (§9).
- Sunbay handles scheduling, incremental watermarking, paging and retries. Your side only serves data.
- Both paid and unpaid invoices must be exposed; how far back the history goes is agreed during onboarding (§10).

### 4.2 List invoices

```
GET {baseUrl}/invoices?modifiedSince=2026-01-24T09:30:00Z&pageSize=100
Authorization: <see §7.2>
Accept: application/json
```

Query parameters (all optional):

| Parameter | Type | Semantics |
|---|---|---|
| `modifiedSince` | ISO-8601 UTC timestamp | Return only invoices with `lastModifiedAt >= modifiedSince` (**inclusive**). When omitted, return a **full snapshot**: all `Open` / `PartiallyPaid` invoices plus `Paid` / `Cancelled` ones within the agreed history window (§10). |
| `cursor` | opaque string | Continuation token copied from the previous response's `nextCursor`. |
| `pageSize` | integer | Maximum items per page. Default `100`; the server may cap it (suggested cap `500`). |

Response `200 OK`, `application/json`:

```json
{
  "items": [ { ...invoice objects exactly as defined in §3... } ],
  "nextCursor": "eyJsYXN0TW9kaWZpZWRBdCI6IjIwMjYtMDEt...",
  "totalCount": 1234
}
```

- `items` - full invoice objects (§3.9 shape), field names 1:1.
- `nextCursor` - `null` or absent on the last page; otherwise Sunbay passes it back **unchanged** in the next request. **Cursor-based pagination is recommended** - offset/page-number paging drops or duplicates rows when data changes mid-crawl. A non-binding implementation hint: keyset pagination ordered by `(lastModifiedAt, externalInvoiceId)` ascending, with the cursor encoding the last-seen pair. Cursors only need to stay valid for the duration of one crawl (hours).
- `totalCount` - optional, informational.
- **Ordering:** ascending by `lastModifiedAt` with a stable tie-break, so an interrupted crawl can resume safely.

**Incremental fetching (watermarking).** After each completed crawl Sunbay stores the highest `lastModifiedAt` seen, and polls next with `modifiedSince = watermark - small overlap` (a few minutes). Any duplicates this causes are harmless - `externalInvoiceId` idempotency (§8) makes re-processing a safe update. Your obligations: every data change bumps `lastModifiedAt`, the filter is inclusive, and timestamps are UTC.

### 4.3 Invoice PDF (optional)

> Implement this endpoint **only if** invoice documents should be attached to reminder emails. If you do not need attachments, skip it - the integration works fully without PDFs.

```
GET {baseUrl}/invoices/{externalInvoiceId}/pdf
```

- `200 OK` with `Content-Type: application/pdf` and the binary document (`Content-Disposition` filename optional).
- `404` when the id is unknown or the PDF is not yet available - Sunbay retries later.
- The path parameter is the URL-encoded `externalInvoiceId`.
- Sunbay fetches each PDF **once** - on first ingestion, and again after a corrective document - so PDFs are never shipped proactively or repeatedly.
- Size guideline: up to ~10 MB per document (confirmed during onboarding).

### 4.4 Single invoice (optional, recommended)

```
GET {baseUrl}/invoices/{externalInvoiceId}
```

Returns `200 OK` with one invoice object (§3), or `404` if unknown. Used for spot re-fetches and joint debugging; not required for the integration to work.

### 4.5 Snapshots, increments & cancellations

- Regular polls are **incremental** (`modifiedSince`). In addition, Sunbay may periodically run a **full-snapshot** crawl (no `modifiedSince`) to reconcile state - e.g. nightly or weekly (§9).
- **Cancellations must stay visible.** A cancelled or deleted invoice must remain retrievable through the API as a *tombstone*: returned with `status = "Cancelled"` and an updated `lastModifiedAt`. It must **not** silently disappear from results - otherwise Sunbay would keep chasing a debt that no longer exists. If the source system hard-deletes records, the API layer must still expose the tombstone.
- Safety net: during full-snapshot reconciliation, open invoices missing from the snapshot are flagged and handled per the onboarding agreement (§10).

### 4.6 Errors & availability

- Status codes: `400` invalid parameters · `401`/`403` authentication failures · `429` rate limited (with `Retry-After`, which Sunbay honours) · `5xx` server errors, which Sunbay retries with backoff (§8).
- Error bodies: JSON in the form `{ "error": { "code": "...", "message": "..." } }` is recommended, not mandated.
- Responses should complete within ~30 seconds; prefer lowering the page size over risking timeouts.
- Authentication: one of the options in §7.2, chosen per your security policy.
- No hard SLA is required - a brief outage only delays the next successful poll. Data freshness should roughly match the agreed poll interval (§9).

### 4.7 Example exchange

```
GET /sunbay/v1/invoices?modifiedSince=2026-01-24T09:30:00Z&pageSize=100
Authorization: Bearer eyJhbGciOi...
Accept: application/json
```

```json
{
  "items": [
    { ...the invoice object from §3.9... }
  ],
  "nextCursor": null,
  "totalCount": 1
}
```

And - only when PDF attachments are enabled (§3.7):

```
GET /sunbay/v1/invoices/ERP-2026-INV-000123/pdf
Authorization: Bearer eyJhbGciOi...

HTTP/1.1 200 OK
Content-Type: application/pdf

%PDF-1.7 ...binary...
```

---

## 5. Option B: Push to Sunbay

Use this option when your environment cannot expose an inbound endpoint: connectivity is one-way, your system can send data out to Sunbay over the public internet, and Sunbay cannot initiate connections back.

> The endpoints below are a **proposal** to illustrate the intended shape. The exact paths, request/response bodies, acknowledgment mechanism, and retry behaviour are finalised together during onboarding - they depend on how your delivery job is built and on whether you want to read back per-invoice results (§5.5).

### 5.1 Per-invoice push (primary)

For each invoice, your system makes **one call** to Sunbay carrying the invoice's structured data (§3). When PDF attachments are enabled (§3.7), the PDF travels **together** with the data in the same call as `multipart/form-data`; without PDFs, the push is a plain JSON request. Invoices are sent one after another on your interval - each call is small, so a standard HTTPS request is sufficient.

`POST /api/ingest/invoices`:

```
POST /api/ingest/invoices
Authorization: <see §7.3>
Content-Type: multipart/form-data; boundary=----sunbay

------sunbay
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

{
  "tenantCode": "ACME-PL",
  "syncMode": "Incremental",
  "invoice": { ...the invoice object from §3.9... }
}
------sunbay
Content-Disposition: form-data; name="pdf"; filename="FV-2026-01-0123.pdf"
Content-Type: application/pdf

%PDF-1.7 ...binary...
------sunbay--
```

The `pdf` part is **optional** - include it only when PDF attachments are enabled (§3.7).

Sunbay responds with an HTTP status indicating receipt (`2xx` accepted, `4xx` for a malformed/invalid request). The precise success/error body is part of the acknowledgment design (§5.5).

### 5.2 Bulk data-only (backfill)

`POST /api/ingest/bulk` - a single compressed file (`.zip` of CSV or JSON Lines), **data only, no PDFs** - for initial historical backfill or very high volumes. The same logical fields from §3 apply, one record per invoice. This is an alternative to issuing thousands of individual calls for a first load.

### 5.3 Sync modes

Which invoices to send each cycle is chosen during onboarding:

- **Full snapshot** - each cycle, send **all currently-open invoices** (plus recently-paid ones, so settlements are reflected). Simple and self-healing - corrections, cancellations and payments are naturally picked up because the complete current picture is resent.
- **Incremental** - each cycle, send **only invoices created or changed since the previous successful sync** (including those whose status changed to paid). Lighter, but **deletions/cancellations in the source system will not appear as a "change"** - so an **explicit cancellation signal** is required (`status = Cancelled`, or a dedicated cancel call), otherwise Sunbay would keep chasing a debt that no longer exists.

### 5.4 Optional "sync session" framing

To bound a full snapshot and give a natural place to report results, calls in one cycle may be grouped in a session: a **begin** call returns a `sessionId`, per-invoice calls reference it, and a **complete** call closes the cycle. This is optional and subject to the same "to be agreed" note above.

### 5.5 Acknowledgment, errors & duplicates - to be agreed

Because Sunbay cannot call your system in this option, any per-invoice or per-batch result you need (accepted / rejected with reasons / duplicate) must be **read back by your delivery job**. Whether you want this at all, what response bodies Sunbay returns, how validation errors and duplicates are signalled, and how this ties into retries (§8) - all of this is agreed during onboarding.

---

## 6. Data Formats & Conventions

These conventions apply to **both options** - to your API responses in Option A, and to pushed payloads/files in Option B.

| Aspect | Convention |
|---|---|
| **Text encoding** | **UTF-8** for all text and for any CSV/JSON content. ⚠️ Some ERP exports default to regional encodings (e.g. Windows-125x) - convert to UTF-8. |
| **Dates** | ISO-8601 calendar dates: `YYYY-MM-DD` (e.g. `2026-01-24`). |
| **Timestamps** | ISO-8601 in **UTC** with `Z` (e.g. `2026-01-24T09:30:00Z`). |
| **Decimal separator** | **Dot** (`.`). No thousands separators. ⚠️ Locales using a decimal comma must be normalised (`1230,00` → `1230.00`). |
| **Currency** | ISO 4217 three-letter code. |
| **Phone** | International format with country code, e.g. `+48512345678`. |
| **Booleans** | `true` / `false`. |
| **Missing values** | Omit the field or send `null` - do not send empty placeholder strings for numeric/date fields. |

---

## 7. Security & Authentication

### 7.1 Transport

- All communication - in either direction - uses **HTTPS / TLS 1.2+**.

### 7.2 Option A - securing your API

You decide how your endpoint authenticates Sunbay; any of the following works, chosen per your security policy during onboarding:

- **API key** - a key you issue to Sunbay, sent in a header (e.g. `x-api-key`). You control issuance and rotation.
- **OAuth 2.0 client credentials** - you provide a token endpoint and a client id/secret; Sunbay exchanges them for short-lived bearer tokens and caches tokens until expiry.
- **Mutual TLS (mTLS)** - Sunbay presents a client certificate you trust.

Additionally:

- **IP allowlisting is feasible in this option**: Sunbay calls from a stable, published set of egress IP addresses, which you may allowlist at your perimeter.
- **Tenant identification is implicit** - the base URL and credentials identify the client on both sides; no `tenantCode` field is used in Option A.
- Sunbay stores the credentials you issue in a secrets store and supports rotation.

### 7.3 Option B - authenticating to Sunbay

The method is agreed during onboarding and aligned with your security policy:

- **API key** - a dedicated key issued by Sunbay, sent in a header (e.g. `x-api-key`). Simple; Sunbay controls issuance and rotation.
- **OAuth 2.0 client credentials** - your system exchanges a client id/secret for a short-lived bearer token.
- **Mutual TLS (mTLS)** - your system is identified by a client certificate.

**Tenant identification:** each call carries a `tenantCode` together with the credential; Sunbay maps the pair to a single tenant. Data is isolated per tenant; a credential can only write data for its own tenant. For a single-client integration the `tenantCode` is a fixed constant provided at onboarding.

If your outbound egress uses static IP addresses, allowlisting on Sunbay's side can additionally be agreed; otherwise authentication relies on the credential.

### 7.4 Data protection

- Encrypted in transit (TLS) and at rest on the Sunbay side.
- Sensitive values (keys, tokens, certificates) are never logged.
- Credentials are exchanged securely and can be rotated on either side.

---

## 8. Reliability

**Idempotency (both options).** `externalInvoiceId` is the key: re-sending or re-fetching the same invoice is a safe **update**, never a duplicate. An invoice's `externalInvoiceId` must never change between syncs, edits or retries.

**Option A - Sunbay retries**

- Transient failures, timeouts and `5xx` responses are retried with exponential backoff; `429` with `Retry-After` is honoured.
- Your obligations: stable identifiers, an **inclusive** `modifiedSince` filter, `lastModifiedAt` updated on every data change, cursors valid for the duration of a crawl, and cancellation tombstones (§4.5).
- Brief outages are unproblematic - they only delay the next successful poll.
- Ordering of corrections is handled by Sunbay internally: if a corrective document appears before its original (e.g. across page boundaries), it is accepted and linked once the original arrives.

**Option B - your delivery job retries**

- On a transient/network/`5xx` failure, retry with backoff; a malformed/`4xx` request should not be blindly retried.
- The precise retry contract ties into the acknowledgment design (§5.5) and is agreed during onboarding.
- If a correction can be pushed before its original, the handling (hold / accept-then-link) is agreed during onboarding.

---

## 9. Scheduling & Volume

**Option A**

- Sunbay polls incrementally on an agreed schedule - typically every 15-60 minutes - plus an optional periodic full snapshot (e.g. nightly or weekly) for reconciliation.
- PDF fetches (if enabled) happen once per new or corrected invoice, with bounded concurrency.
- Tell us your **rate limits** and maintenance windows - Sunbay stays within them and honours `429` / `Retry-After`.
- Please share expected **daily and peak volumes** (e.g. month-end) so page size and poll frequency can be sized sensibly.

**Option B**

- Your system pushes on a configurable schedule (e.g. every 15 minutes, hourly, or daily), agreed during onboarding.
- Call volume scales with the number of invoices per cycle; for very high volumes, the bulk path (§5.2) is recommended for backfill.
- Sunbay may apply rate limiting; limits and any `Retry-After` behaviour are agreed so your delivery pace fits within them.

---

## 10. Points to Confirm During Onboarding

> **Note:** these points are for the implementation/rollout phase itself - not something to resolve before reviewing or sharing this document. We work through them together when the integration is being built.

**Common (both options)**

1. **Stable identifiers** - does the source system expose a stable, unique id per **invoice** and per **customer** that survives edits? What are they?
2. **Corrections** - how does the source system represent corrective documents, and are their amounts the **new corrected totals**, the **delta**, or "before/after"? (§3.1.2)
3. **Partial payments** - can `amountPaid` / `amountOutstanding` be provided, or only a binary paid flag?
4. **Document types** - which document kinds exist and which are collectible; mapping of any kinds not listed in §3.1.1. Should proformas be excluded?
5. **Multi-company** - can one installation hold several legal entities/sellers? If so, how is the seller disambiguated?
6. **Currencies** - are multi-currency invoices expected?
7. **Volumes** - expected daily and peak invoice counts.
8. **Formats** - confirm UTF-8, dot decimals, ISO-8601 (including time-zone handling for dates), and phone numbers with country code can be guaranteed (§6).
9. **PDF attachments - yes or no?** Should Sunbay attach invoice documents to reminder emails? Only if yes: PDF availability, maximum size, and PDFs for corrective documents (§3.7).

**Option A**

10. API base URL, credential exchange and rotation procedure; chosen authentication method (§7.2).
11. Your rate limits and maintenance windows; Sunbay egress IPs for allowlisting.
12. Initial load depth - how far back paid invoices are exposed (e.g. all open, plus paid within N months).
13. `lastModifiedAt` semantics - which changes bump it, and with what precision?
14. Test/sandbox environment availability.
15. Poll schedule - incremental interval and full-snapshot cadence (§9).

**Option B**

16. **Acknowledgment** - do you want to read back per-invoice/per-batch results? If so: response bodies, error/duplicate signalling, and the retry tie-in (§5.5).
17. Push frequency and backfill mechanics (bulk file vs per-invoice) (§5.2, §9).
18. Chosen authentication method (§7.3) and its feasibility from your environment; whether your egress IPs are static (optional allowlisting).
