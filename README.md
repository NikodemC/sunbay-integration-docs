# ERP → Sunbay Invoice Data Integration Guide

This document specifies the **invoice data Sunbay needs to obtain** from your ERP / source system, and **the two ways that data can be exchanged**, so that Sunbay can run debt collection and receivables analytics on your behalf.

It is written for the **IT team building the integration** between the ERP landscape and Sunbay.

Two integration options are supported:

- **Option A (recommended)** - you host a small read-only API; **Sunbay pulls** invoices on its own schedule (§4).
- **Option B** - your system **pushes** invoices to Sunbay's ingestion endpoints (§5).

The **data model (§3) is the core of this document** and is identical for both options. Sections §4 and §5 describe the two exchange mechanisms; §6-§9 cover formats, security, reliability and scheduling. A number of protocol details are deliberately left **open for onboarding (§10)**.

> **On flexibility.** This is the specification to implement, not a loose suggestion. Where a field or detail genuinely cannot be met by your source system, we can align it together during onboarding (§10). Please implement the contract as written and flag the specific points that do not fit.

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
- **Paid invoices** power analytics and reporting across your receivables.

Optionally, the **PDF** of an invoice can be exchanged as well - it is needed **only** when invoice documents should be attached to reminder emails (§3.7).

In both integration options the ERP remains the **system of record**. Sunbay never writes back to the ERP - the integration is read-only from the ERP's perspective.

**In scope for this document**

- The exact data fields Sunbay needs per invoice and per customer (§3).
- The two exchange mechanisms: a client-hosted API that Sunbay polls (§4), or pushes to Sunbay's ingestion endpoints (§5).
- Data formats (§6), authentication (§7), reliability rules (§8), scheduling and volumes (§9).

---

## 2. Integration Options

### 2.1 Option A - client-hosted API, Sunbay pulls (recommended)

You expose a small, **read-only HTTPS API** over the ERP data (specified in §4). Sunbay calls it on a schedule it controls: incremental polls for new and changed invoices, plus periodic full snapshots for reconciliation.

Why this is the recommended option:

- You implement **one to three small read-only endpoints** (an invoice endpoint, plus a PDF endpoint and a single-invoice endpoint where §4.3 and §4.5 call for them) - no scheduler, no retry logic, no outbound delivery pipeline on your side.
- Sunbay owns scheduling, backfill, retries and pacing - and can adapt them without any change on your side.
- Cancellations and corrections propagate naturally: Sunbay simply observes the current state of your data.
- Documents stay authoritative in your system: PDFs are fetched on demand and never stored by Sunbay (unlike push, where Sunbay must keep a copy).

### 2.2 Option B - push to Sunbay

Your system pushes each invoice to Sunbay's ingestion endpoints (§5), one call per invoice, on a schedule you run. Choose this option when connectivity is a hard constraint - i.e. no inbound endpoint may be exposed from your network, and your systems can only **send data out** over the public internet.

### 2.3 Comparison

| Aspect | Option A - Sunbay pulls | Option B - you push |
|---|---|---|
| Who initiates | Sunbay (scheduled polls) | Your system (own scheduler) |
| You implement | Read-only invoice endpoint (+ PDF and single-invoice endpoints where §4.3 and §4.5 apply) | A delivery job: per-invoice pushes, retries, backoff |
| Scheduling & retries | Owned by Sunbay | Owned by you |
| Cancellations | Visible as `Cancelled` tombstones in API responses (§4.5) | Require an explicit cancellation signal in incremental mode (§5.3) |
| Backfill / first load | Sunbay crawls the history via the same API | Bulk file upload or many individual pushes (§5.2) |
| PDFs (if enabled) | Fetched by Sunbay on demand at each reminder send (no copies kept) | Shipped by you with each push and stored by Sunbay |
| Tenant identification | Implicit - credentials and base URL are client-specific (§7.2) | `tenantCode` field in every payload (§7.3) |
| Result reporting | HTTP responses of your API | Acknowledgment design to be agreed (§5.5) |

### 2.4 Common to both options

- The **data model (§3)** and **formats (§6)** are identical.
- **Idempotency** is keyed on `invoiceId` (§8) - re-delivery or re-fetch of the same invoice is a safe update, never a duplicate.
- Both **paid and unpaid** invoices are in scope.
- **PDFs are optional** in both options - they are needed only for email attachments (§3.7).

---

## 3. Data Model

The **most important section**. Each invoice, pulled or pushed, carries the fields below.

An invoice is a JSON object carrying the invoice-level fields at the **top level** plus the nested `customer` object (§3.4) and, where it is needed, `seller` (§3.5). Debtor data is embedded with each invoice (denormalized) - there is no separate customer feed. Field names inside the nested objects are **unprefixed** (`customer.name`, `seller.name`); the complete shape is shown in §3.9.

**Required** column legend: **Yes** = mandatory · **Rec.** = recommended (strongly preferred) · **Opt.** = optional · **Cond.** = conditional.

### 3.1 Invoice identification

| Field | Type | Required | Description |
|---|---|---|---|
| `invoiceId` | string | **Yes** | Stable, globally-unique identifier of the invoice **in the source system**. Used to recognise the same invoice across syncs (deduplication / update key). Must be **stable** - the same invoice must always carry the same id, even after edits. |
| `invoiceNumber` | string | **Yes** | Human-readable invoice number (e.g. `FV/2026/01/0123`). Shown to the debtor in reminders. **Not required to be unique** - see §3.1.3. |
| `documentType` | enum | **Yes** | Kind of document - see §3.1.1. |
| `correctedInvoiceId` | string | **Cond.** | On an adjusting document (`CorrectiveInvoice`, `CreditNote`, `DebitNote`) that adjusts a specific invoice: the `invoiceId` of that invoice. A standalone note that adjusts nothing - late-payment interest, a penalty charge - carries no `correctedInvoiceId` and stands as a receivable of its own. Always point at the original invoice, never at an earlier correction of it. |
| `issueDate` | date | **Yes** | Date the invoice was issued. |
| `dueDate` | date | **Cond.** | Payment due date - when the invoice becomes collectible. Required on every document except a `CreditNote`, where `null` is accepted. |
| `paymentTermDays` | integer | **Opt.** | Payment term in days, if available. |

#### 3.1.1 `documentType` values

| Value | Meaning | Chased by Sunbay? |
|---|---|---|
| `Invoice` | Standard (VAT) invoice | Yes |
| `CorrectiveInvoice` | Corrective / adjustment invoice | Yes |
| `AdvanceInvoice` | Advance payment invoice | Yes |
| `FinalInvoice` | Final invoice | Yes |
| `Proforma` | Pro forma invoice | **No** - not a legal receivable |
| `CreditNote` | Credit note | **No** - a negative document; its open amount offsets what the debtor owes |
| `DebitNote` | Debit note | Yes |

"Yes" means the same thing everywhere: chased **while its own `amountOutstanding` is positive** (§3.1.2). Proformas never are, whatever the amounts, and a credit note cannot be - a refund exceeding the note itself is a bookkeeping error, not a case this contract covers.

> If your source system uses other document kinds, list them during onboarding so we can map them (§10).

#### 3.1.2 Corrections

`CorrectiveInvoice`, `CreditNote` and `DebitNote` arrive as **separate records**, each with its own `invoiceId`, pointing through `correctedInvoiceId` at the invoice they adjust - where there is one. The original's `amountNet` / `amountVat` / `amountGross` are never rewritten; only its open amount moves.

**Their own `amountNet` / `amountVat` / `amountGross` are the difference they introduce, signed.** Negative reduces what the debtor owes, positive increases it. If your system stores corrections as new totals or as "before/after", derive the difference in the integration layer (the representation is confirmed at onboarding, §10). Never send an absolute value and leave the direction to be read off the document type or the accounting side - a `CorrectiveInvoice` goes both ways.

**Sunbay reads open amounts; it never adds them up.** `amountOutstanding` is what that one document still has open in your source system. Records with a positive one are chased, and nothing is recomputed from correction chains, so a credit already applied cannot be applied twice. The consequence: any reduction the debtor is owed must **already be inside** the `amountOutstanding` of the document being chased.

| If your source system | Expose |
|---|---|
| Clears credit notes against their originals | Nothing extra - the cleared state is already right |
| Keeps both as open items | The clearing, applied by the integration layer: the original's `amountOutstanding` drops by the credit, and whatever it cannot absorb stays on the credit note as a negative open amount (a refund you owe) |
| Issues credit notes with no `correctedInvoiceId` (rebate, goodwill) | The note as it is, negative. Reminders ignore it until your source system allocates it - Sunbay never allocates credits |

Positive adjustments run the other way: a debit note or an increasing correction is a receivable in its own right, chased on its own `amountOutstanding`.

`correctedInvoiceId` also groups the two records for reminders and analytics, so the referenced original must reach Sunbay too - see §4.5 (Option A) and §5.3 (Option B).

#### 3.1.3 `invoiceNumber` is not a unique key

Only `invoiceId` is unique. One accounting document may arrive as **several records sharing an `invoiceNumber`** - typically when the source system keys documents per line, each line with its own `invoiceId`, amounts and sometimes due date.

That is accepted, but the debtor still holds one document with one number while Sunbay chases several receivables carrying it. Tell us at onboarding if your data works this way (§10), so reminder content and payment matching are set up for it.

### 3.2 Amounts & currency

| Field | Type | Required | Description |
|---|---|---|---|
| `currency` | string (ISO 4217) | **Yes** | e.g. `PLN`, `EUR`. |
| `amountNet` | decimal | **Yes** | Net amount. |
| `amountVat` | decimal | **Yes** | VAT amount. |
| `amountGross` | decimal | **Yes** | Gross total of this document. Signed on adjusting documents (§3.1.2). |
| `amountPaid` | decimal | **Yes** | Amount settled against this document by payments (`0` if none). Enables partial-payment handling. **Never clamped** - see overpayments below. |
| `amountOutstanding` | decimal | **Yes** | **The open amount of this document in your source system, right now.** This is the authoritative value and what collection chases. It is normally `amountGross - amountPaid`, but not always: netting - in either direction - changes it without touching `amountPaid` (§3.1.2), and cancelled documents report `0` (§3.3). |

**Signs.** `amountNet` / `amountVat` / `amountGross` are non-negative on ordinary documents, signed on adjusting ones (§3.1.2). `amountPaid` takes the sign of its own document, so a refund against a credit note is negative. `amountOutstanding` may be negative on anything - an overpaid invoice, a credit note still awaiting refund - and only a **positive** one is chased.

**Overpayments.** Receiving more than the document still owes must **not** be clamped: `amountPaid` keeps the real figure, `amountOutstanding` goes negative by the surplus, `status` is `Paid`. An invoice paid at face value after a credit was cleared against it (§3.1.2) counts as one. Capping `amountPaid` would delete the surplus from the feed.

### 3.3 Status & lifecycle

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | enum | **Yes** | `Open` · `PartiallyPaid` · `Paid` · `Cancelled`. |
| `paidDate` | date | **Cond.** | Date the document was fully settled - by payment, or by the netting that closed it. Required when `status = Paid`. |
| `isBlockedForCollection` | boolean | **Rec.** | `true` if Sunbay must **not** chase this invoice (dispute, legal hold, internal block). No such concept in the source system? **Omit the field** rather than sending `false` everywhere: absent means *nothing is known*, `false` asserts the invoice may be chased. Either way Sunbay chases it. |
| `lastModifiedAt` | timestamp | **Yes** (Option A) / **Rec.** (Option B) | When the record **as delivered to Sunbay** last changed (ISO-8601 UTC). Drives incremental fetching in Option A (§4.2), so **any** change bumps it - status, amounts, payments, cancellation, a credit cleared against it (§3.1.2), correction linkage - including values the integration layer derives while the source record sits untouched. |

**Status must follow from the amounts**, on every document kind. `amountGross` says which direction the document runs in; `amountOutstanding` is read against that sign.

| Condition | `status` |
|---|---|
| Nothing settled yet: `amountPaid = 0`, and `amountOutstanding` has the same sign as `amountGross` | `Open` |
| Partly settled: `amountPaid ≠ 0`, and `amountOutstanding` has the same sign as `amountGross` | `PartiallyPaid` |
| Nothing open: `amountOutstanding = 0`, or its sign is opposite to `amountGross` (overpaid) | `Paid` (with `paidDate` set) |
| Document cancelled or voided in the source system | `Cancelled`, whatever the amounts |

A document with `amountGross = 0` - a correction to descriptive data only - has nothing to settle: `amountOutstanding = 0`, `status = Paid`, `paidDate = issueDate`.

Should a record still arrive self-contradictory (`PartiallyPaid` with nothing outstanding, say), **the amounts decide**: only a positive `amountOutstanding` is chased. `Cancelled` and `isBlockedForCollection = true` override the amounts and stop collection outright.

**Cancelled documents.** `amountGross` stays nominal and `amountPaid` stays truthful - usually `0`, but a payment booked before the document was voided remains visible, as a refund for your side to handle. `amountOutstanding` is forced to `0` regardless of the ledger, and `paidDate` is `null`: nothing is collectible on a cancelled document. The record must still reach Sunbay - as a tombstone in Option A (§4.5), or through the explicit cancellation signal in Option B (§5.3).

### 3.4 Debtor (customer)

Delivered as a **nested `customer` object** on every invoice. This is the party Sunbay contacts, so the object itself is required.

| Field (inside `customer`) | Type | Required | Description |
|---|---|---|---|
| `id` | string | **Yes** | Stable, unique identifier of the customer in the source system. |
| `name` | string | **Yes** | Debtor name (company or person). |
| `taxId` | string | **Rec.** | Tax identifier (e.g. VAT ID / NIP). One consistent format across the whole feed (§6). |
| `email` | string | **Rec.** | Primary email. Required for email reminders. |
| `emailCcs` | string[] | **Opt.** | **List** of additional CC email addresses. |
| `phone` | string | **Opt.** | Phone number in **international format including the country code**, e.g. `+48512345678`. Required for SMS reminders. |
| `address` | string | **Rec.** | Postal address. |
| `countryCode` | string (ISO 3166-1) | **Yes** | e.g. `PL`. |
| `communicationLanguage` | string (ISO 639-1) | **Opt.** | Preferred language for reminders, e.g. `pl`, `en`, `de`. Sunbay selects the reminder template language per debtor; when absent, the client-wide default is used. |
| `customFields` | object | **Opt.** | Arbitrary key-value pairs at **customer** level (see §3.6). |

### 3.5 Seller & payment

A **nested `seller` object**, needed only where the issuing entity or its bank account varies between invoices.

- **One entity, one bank account** - omit `seller` entirely. Sunbay uses the default configured for your client at onboarding (§10), so constant data is not repeated on every record.
- **Several entities, or a bank account that varies** (separate currency accounts, for instance) - send `seller` on the invoices it applies to.
- A `seller` that is present **replaces the default in full**; fields are not merged one by one. So include the bank account whenever you include the object.
- No `seller` and no configured default means no account to quote, and reminders cannot go out.

| Field (inside `seller`) | Type | Required | Description |
|---|---|---|---|
| `name` | string | **Opt.** | Issuing entity name. |
| `taxId` | string | **Opt.** | Seller tax identifier. |
| `address` | string | **Opt.** | Issuing entity postal address. Useful for formal reminders and multi-entity installations. |
| `bankAccount` | string | **Yes** | Bank account the debtor should pay into (IBAN/NRB). Included in reminders. Required whenever the object is sent. |

### 3.6 Custom fields & references

| Field | Type | Required | Description |
|---|---|---|---|
| `customFields` | object | **Opt.** | Arbitrary key-value pairs at **invoice (top) level**; the same is available per customer (§3.4). Send any extra attribute that may be useful - segment, region, contract code, cost centre. |
| `externalReference` | string | **Opt.** | Any additional reference useful for reconciliation. |

### 3.7 PDF (optional)

> **PDFs are optional.** A PDF is needed **only if** you want Sunbay to attach the invoice document to reminder emails. If you do not need attachments, skip PDFs entirely - the integration is fully functional without them.

When they are enabled:

- **Option A:** Sunbay fetches each PDF from your endpoint (§4.3) at every reminder send, and keeps no copies.
- **Option B:** the PDF travels with the invoice data in the same push (§5.1).

One PDF per invoice; availability, maximum size and PDFs for corrective documents are settled at onboarding (§10).

### 3.8 Line items (optional)

Line items are **optional but valuable**: richer analytics, more informative reminders. The invoice works without them, and where PDFs are exchanged the PDF stays the authoritative document. On an adjusting document the lines are signed like its header (§3.1.2).

If provided, `lineItems` is an array where each line carries:

| Field | Type | Required (within a line) | Description |
|---|---|---|---|
| `name` | string | **Yes** | Description of the goods/service. |
| `quantity` | decimal | **Yes** | Quantity. |
| `unit` | string | **Opt.** | Unit of measure, e.g. `pcs`, `hours`. |
| `unitPriceNet` | decimal | **Yes** | Net unit price. |
| `discountPercent` | decimal | **Opt.** | Line discount in percent, e.g. `10.0`. The line totals below are already net of this discount - it is informational. |
| `vatRate` | decimal | **Rec.** | VAT rate in percent, e.g. `23.0`. |
| `vatAmount` | decimal | **Opt.** | VAT amount for the line. |
| `amountNet` | decimal | **Rec.** | Net total for the line. |
| `amountGross` | decimal | **Yes** | Gross total for the line. |

### 3.9 Example invoice object

The same invoice object is used in both options: it is the item shape returned by your API (Option A, §4.2) and the `invoice` part of a push payload (Option B, §5.1). It is shown with `seller` present; a single-entity feed leaves that object out (§3.5).

```json
{
  "invoiceId": "ERP-2026-INV-000123",
  "invoiceNumber": "FV/2026/01/0123",
  "documentType": "Invoice",
  "correctedInvoiceId": null,
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
    "name": "ACME Sp. z o.o.",
    "taxId": "5213001234",
    "address": "ul. Handlowa 5, 00-002 Warszawa",
    "bankAccount": "PL61109010140000071219812874"
  },

  "customer": {
    "id": "ERP-CUST-10001",
    "name": "Kowalski Handel Sp. z o.o.",
    "taxId": "7010001234",
    "email": "ksiegowosc@kowalski.pl",
    "emailCcs": ["zarzad@kowalski.pl", "biuro@kowalski.pl"],
    "phone": "+48512345678",
    "address": "ul. Przykładowa 12, 00-001 Warszawa",
    "countryCode": "PL",
    "communicationLanguage": "pl",
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

The contract Sunbay's fetcher codes against. **Host and base path are yours** - a versioned prefix is recommended, e.g. `{baseUrl} = https://api.example.com/sunbay/v1`. Below that base URL, implement paths, parameters and response shapes as specified; deviations can be discussed at onboarding.

### 4.1 Overview & responsibilities

- You host a **read-only** HTTPS API exposing invoices in the shape defined in §3. Sunbay polls it on an agreed schedule (§9).
- Sunbay handles scheduling, incremental watermarking, paging and retries. Your side only serves data.
- Both paid and unpaid invoices must be exposed; how far back the history goes is agreed during onboarding (§10).

### 4.2 List invoices

```
GET {baseUrl}/invoices?modifiedSince=2026-01-24T09:30:00Z&modifiedUntil=2026-01-24T10:00:00Z&page=1&pageSize=100
Authorization: <see §7.2>
Accept: application/json
```

Query parameters (Sunbay may omit any of them; all must be supported):

| Parameter | Type | Semantics |
|---|---|---|
| `modifiedSince` | ISO-8601 UTC timestamp | Lower bound, **inclusive**: return only invoices with `lastModifiedAt >= modifiedSince`. When omitted, return a **full snapshot**: all `Open` / `PartiallyPaid` invoices, `Paid` / `Cancelled` ones within the agreed history window (§10), and - when route (a) of §4.5 applies - any invoice referenced by `correctedInvoiceId` from a record in the snapshot, regardless of its age. |
| `modifiedUntil` | ISO-8601 UTC timestamp | Upper bound, **exclusive**: `lastModifiedAt < modifiedUntil`. Set to the instant the crawl started, so the result set stays frozen for its duration. One extra condition in your query. |
| `page` | integer | 1-based page number. Default `1`. |
| `pageSize` | integer | Maximum items per page. Default `100`; you may cap it (suggested cap `500`) - echo the cap you applied in the response. |

Response `200 OK`, `application/json`:

```json
{
  "items": [ { ...invoice objects exactly as defined in §3... } ],
  "page": 1,
  "pageSize": 100,
  "totalCount": 1234
}
```

- `items` - full invoice objects (§3.9 shape), field names 1:1.
- `page` / `pageSize` - `page` echoes the request; `pageSize` is the size **actually applied**, so a capped request reports the cap rather than what was asked for. Sunbay walks pages until one returns fewer items than that applied size (or `page * pageSize` reaches `totalCount`) - echoing the requested value instead would end the crawl after the first capped page.
- `totalCount` - matching invoices in total. **Recommended**, so Sunbay can size the crawl; without it, paging just stops on the first short page.
- **Ordering:** `(lastModifiedAt, invoiceId)` ascending. With `modifiedUntil` freezing the set, this makes a crawl deterministic - paging can neither skip nor repeat a row, and an interrupted crawl resumes on the same boundaries.
- **Why page numbers and not opaque cursors:** they cost you a `LIMIT`/`OFFSET`, and `modifiedUntil` removes their usual danger - a record modified mid-crawl leaves the window instead of shifting position inside it and pushing its neighbour past a page boundary unread. Without `modifiedUntil` the set moves under you; `invoiceId` idempotency (§8) still makes duplicates harmless and the full-snapshot crawl (§4.5) reconciles the rest, but a skipped change then stays stale until that snapshot.

**Incremental fetching (watermarking).** Sunbay picks a crawl instant `T`, calls with `modifiedUntil = T` and `modifiedSince = previous watermark - a few minutes of overlap`, then stores `T` as the new watermark once the crawl completes. The upper bound is what makes that watermark trustworthy: everything below `T` has been served, and a record modified mid-crawl simply belongs to the next one. Snapshots carry `modifiedUntil` the same way and seed the watermark for the polls after them. The overlap re-reads a few records - harmless under `invoiceId` idempotency (§8). Your side owes: `lastModifiedAt` bumped on every change you expose, `modifiedSince` inclusive, `modifiedUntil` exclusive, timestamps in UTC.

### 4.3 Invoice PDF (optional)

> Implement this endpoint **only if** invoice documents should be attached to reminder emails. If you do not need attachments, skip it - the integration works fully without PDFs.

```
GET {baseUrl}/invoices/{invoiceId}/pdf
```

- `200 OK` with `Content-Type: application/pdf` and the binary document (`Content-Disposition` filename optional).
- `404` when the id is unknown or the PDF is not yet available - Sunbay retries later.
- The path parameter is the URL-encoded `invoiceId`.
- Sunbay stores no copies: it fetches the PDF **each time it sends a reminder with the invoice attached**, so the same document may be requested many times over a collection. Nothing is shipped proactively - it is served on request, and the endpoint must stay available for as long as the invoice is chased.
- Size guideline: up to ~5 MB per document (confirmed during onboarding).

### 4.4 Single invoice (recommended)

```
GET {baseUrl}/invoices/{invoiceId}
```

Returns `200 OK` with one invoice object (§3), or `404` if unknown. Used for spot re-fetches and joint debugging.

> Optional in general - **except** when it is the route chosen for reaching corrected originals (§4.5, route **b**). In that case this endpoint is mandatory.

### 4.5 Snapshots, increments, cancellations & corrections

- Regular polls are **incremental** (`modifiedSince`). In addition, Sunbay periodically runs a **full-snapshot** crawl (no `modifiedSince`, still bounded by `modifiedUntil`) to reconcile state - e.g. nightly or weekly (§9).
- **Cancellations must stay visible.** A cancelled or deleted invoice stays retrievable as a *tombstone*: `status = "Cancelled"` with an updated `lastModifiedAt`. It must never just vanish from results, or Sunbay keeps chasing a debt that no longer exists - if the source system hard-deletes, the API layer still exposes the tombstone.
- **The corrected original must stay reachable.** Balances do not depend on it (§3.1.2), but reminders and analytics present the adjusting document together with the invoice in `correctedInvoiceId` - and that original is often years old, long paid, and outside the agreed history window (§4.2). Pick one route at onboarding (§10):
  - **(a)** the API layer keeps referenced originals in scope - an invoice referenced by any record in scope belongs to the snapshot whatever its age or status, and issuing an adjustment bumps its `lastModifiedAt` (correction linkage is a change, §3.3) so it returns in the next poll;
  - **(b)** implement the single-invoice endpoint, which Sunbay calls for originals it has not seen. Here §4.4 is **not optional**.
- **Either route, a changed original must re-enter the feed.** Clearing a credit against an invoice Sunbay already holds (§3.1.2) drops its exposed `amountOutstanding`, and only the incremental poll carries that drop. It is an amount change, so it bumps `lastModifiedAt` (§3.3) in route (b) exactly as in route (a) - §4.4 covers only originals never seen.
- **Safety net.** Open invoices Sunbay expected in a snapshot but did not receive are flagged and handled per the onboarding agreement (§10). Expected means `lastModifiedAt` before that crawl's `modifiedUntil`; anything modified mid-crawl belongs to the next poll, not to the alarm.

### 4.6 Errors & availability

- Status codes: `400` invalid parameters · `401`/`403` authentication failures · `429` rate limited (with `Retry-After`, which Sunbay honours) · `5xx` server errors, which Sunbay retries with backoff (§8).
- Error bodies: JSON in the form `{ "error": { "code": "...", "message": "..." } }` is recommended, not mandated.
- Responses should complete within ~30 seconds; prefer lowering the page size over risking timeouts.
- Authentication: one of the options in §7.2, chosen per your security policy.
- No hard SLA is required - a brief outage only delays the next successful poll. Data freshness should roughly match the agreed poll interval (§9).

### 4.7 Example exchange

```
GET /sunbay/v1/invoices?modifiedSince=2026-01-24T09:30:00Z&modifiedUntil=2026-01-24T10:00:00Z&page=1&pageSize=100
Authorization: Bearer eyJhbGciOi...
Accept: application/json
```

```json
{
  "items": [
    { ...the invoice object from §3.9... }
  ],
  "page": 1,
  "pageSize": 100,
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

**One call per invoice**, carrying its structured data (§3). With PDF attachments enabled (§3.7) the document travels in the same call as `multipart/form-data`; without them it is plain JSON. Calls are small, sent one after another on your interval.

> **PDF storage.** Pushed PDFs are **stored** by Sunbay - it cannot fetch them back from your system when a reminder goes out later. Option A pulls them on demand and stores nothing, which is one reason it is recommended.

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

`POST /api/ingest/bulk` - one compressed file (`.zip` of JSON Lines or CSV), **data only, no PDFs** - instead of thousands of calls for a first load or very high volumes. One record per invoice, fields as in §3:

- **JSON Lines** (preferred) - the §3.9 object unchanged, one per line.
- **CSV** - nested objects flattened to dot-path columns (`customer.name`, `seller.bankAccount`); `lineItems` omitted or JSON-encoded in one column, each `customFields` object JSON-encoded in its own.

### 5.3 Sync modes

Which invoices to send each cycle is chosen during onboarding:

- **Full snapshot** - each cycle, send **all currently-open invoices** (plus recently-paid ones, so settlements are reflected). Simple and self-healing - corrections, cancellations and payments are naturally picked up because the complete current picture is resent.
- **Incremental** - each cycle, send **only invoices created or changed since the previous successful sync** (including those whose status changed to paid). Lighter, but **deletions/cancellations in the source system will not appear as a "change"** - so an **explicit cancellation signal** is required (`status = Cancelled`, or a dedicated cancel call), otherwise Sunbay would keep chasing a debt that no longer exists.

**Corrected originals.** Sunbay cannot fetch anything here, so the delivery job owes it: push an adjusting document (§3.1.2) and the invoice in its `correctedInvoiceId` must have been pushed too. Whether that old, long-paid original shows up as a change depends on your system - netted against it, its open amount moves and it is picked up; kept as a separate open item, it may not move at all. Do not rely on it: push the original alongside the adjustment (re-pushing is a safe update, §8), or confirm at onboarding that the history window covers it (§10).

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
| **Whitespace** | **Trim** leading and trailing whitespace from every text value. ⚠️ Watch for **non-breaking spaces (U+00A0)** inside customer names, addresses and bank accounts - they survive a naive trim and break matching and display; replace them with ordinary spaces. |
| **Tax identifiers** | One **consistent** format across the whole feed: either always with the country prefix (`PL5213001234`) or always without (`5213001234`). No spaces, dashes or dots. |
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

- **Tenant identification is implicit** - base URL plus credentials identify the client; Option A uses no `tenantCode`.
- Credentials you issue are kept in a secrets store and can be rotated.

### 7.3 Option B - authenticating to Sunbay

The method is agreed during onboarding and aligned with your security policy:

- **API key** - a dedicated key issued by Sunbay, sent in a header (e.g. `x-api-key`). Simple; Sunbay controls issuance and rotation.
- **OAuth 2.0 client credentials** - your system exchanges a client id/secret for a short-lived bearer token.
- **Mutual TLS (mTLS)** - your system is identified by a client certificate.

**Tenant identification:** each call carries a `tenantCode` together with the credential; Sunbay maps the pair to a single tenant. Data is isolated per tenant; a credential can only write data for its own tenant. For a single-client integration the `tenantCode` is a fixed constant provided at onboarding.

### 7.4 Data protection

- Encrypted in transit (TLS) and at rest on the Sunbay side.
- Sensitive values (keys, tokens, certificates) are never logged.
- Credentials are exchanged securely and can be rotated on either side.

---

## 8. Reliability

**Idempotency (both options).** `invoiceId` is the key: re-sending or re-fetching the same invoice is a safe **update**, never a duplicate. An invoice's `invoiceId` must never change between syncs, edits or retries.

**Option A - Sunbay retries**

- Transient failures, timeouts and `5xx` responses are retried with exponential backoff; `429` with `Retry-After` is honoured.
- Your obligations: stable identifiers (`invoiceId`, `customer.id`), an **inclusive** `modifiedSince` and **exclusive** `modifiedUntil` filter, `lastModifiedAt` updated on every change in what you expose, a stable ordering by `(lastModifiedAt, invoiceId)` during a crawl, and cancellation tombstones plus reachable corrected originals (§4.5).
- Brief outages are unproblematic - they only delay the next successful poll.
- Ordering of adjusting documents is handled by Sunbay internally: if a correction appears before its original (e.g. across page boundaries), it is accepted and linked once the original arrives.

**Option B - your delivery job retries**

- On a transient/network/`5xx` failure, retry with backoff; a malformed/`4xx` request should not be blindly retried.
- The precise retry contract ties into the acknowledgment design (§5.5) and is agreed during onboarding.
- If a correction can be pushed before its original, the handling (hold / accept-then-link) is agreed during onboarding.

---

## 9. Scheduling & Volume

**Option A**

- Sunbay polls incrementally on an agreed schedule - typically every 15-60 minutes - plus a periodic full snapshot (e.g. nightly or weekly) for reconciliation.
- PDF fetches (if enabled) happen on demand whenever a reminder with an attachment is sent, with bounded concurrency - the same invoice's PDF may be requested more than once over its collection lifecycle.
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

1. **Stable identifiers** - does the source system expose a stable, unique id per **invoice** (`invoiceId`) and per **customer** (`customer.id`) that survives edits? What are they?
2. **Corrections - storage** - how does the source system store adjusting documents: as separate documents with their own open amount, or as edits of the original? Does it clear credit notes against their originals, or keep both as open items - in which case the integration layer has to apply the clearing itself? Do unallocated credit notes occur, and who allocates them? (§3.1.2)
3. **Corrections - sign** - can the signed difference be derived reliably? Does the sign in your system follow the accounting side of the entry, and can the same document kind carry both directions? (§3.1.2)
4. **Split documents** - is one accounting document ever delivered as several receivables sharing an `invoiceNumber` (e.g. keyed per line item)? (§3.1.3)
5. **Partial payments** - can `amountPaid` / `amountOutstanding` be provided, or only a binary paid flag?
6. **Overpayments** - can a payment exceed what is owed on a document, and will the surplus be reported rather than clamped? (§3.2)
7. **Cancelled documents** - can the source system produce the agreed shape (nominal `amountGross`, `amountOutstanding = 0`, truthful `amountPaid`)? (§3.3)
8. **Blocking** - does the source system mark invoices that must not be chased (dispute, legal hold)? If not, how should such cases reach Sunbay - or is the field simply omitted? (§3.3)
9. **Document types** - which document kinds exist and which are collectible; mapping of any kinds not listed in §3.1.1. Proformas are never chased (§3.1.1) - should they be sent at all, for analytics?
10. **Seller and bank account** - one issuing entity or several? Is the bank account constant across invoices? If both are constant, give us the default seller details and account to configure, so `seller` can be omitted from the feed (§3.5).
11. **Currencies** - are multi-currency invoices expected?
12. **Volumes** - expected daily and peak invoice counts.
13. **Formats** - confirm UTF-8, dot decimals, ISO-8601 (including time-zone handling for dates), phone numbers with country code, trimmed text free of non-breaking spaces, and one consistent tax-identifier format (§6).
14. **PDF attachments - yes or no?** Should Sunbay attach invoice documents to reminder emails? Only if yes: PDF availability, maximum size, and PDFs for corrective documents (§3.7).

**Option A**

15. **Corrected originals** - which route from §4.5 applies: referenced originals kept in scope by the API layer, or the single-invoice endpoint (§4.4)?
16. API base URL, credential exchange and rotation procedure; chosen authentication method (§7.2).
17. Your rate limits and maintenance windows.
18. Initial load depth - how far back paid invoices are exposed (e.g. all open, plus paid within N months).
19. `lastModifiedAt` semantics - which changes bump it, and with what precision? Can the query support the `modifiedUntil` upper bound (§4.2)?
20. Test/sandbox environment availability.
21. Poll schedule - incremental interval and full-snapshot cadence (§9).

**Option B**

22. **Corrected originals** - can the delivery job push the invoice referenced by `correctedInvoiceId` alongside the adjusting document, even when the original falls outside the sync window (§5.3)?
23. **Acknowledgment** - do you want to read back per-invoice/per-batch results? If so: response bodies, error/duplicate signalling, and the retry tie-in (§5.5).
24. Push frequency and backfill mechanics (bulk file vs per-invoice) (§5.2, §9).
25. Chosen authentication method (§7.3) and its feasibility from your environment.
