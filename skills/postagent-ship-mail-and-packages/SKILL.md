---
name: postagent-ship-mail-and-packages
description: Print and physically mail a document (PDF/HTML/Markdown/text/DOCX/image) to a US postal address via the PostAgent API. Use when the user wants to send real, physical mail — a letter, notice, invoice, or postcard — to a US recipient. Each send is paid per-call in USDC on Base using the x402 protocol, so it spends real money and is irreversible once submitted.
license: MIT
metadata:
  homepage: https://postagent-api.interpretai.tech
  tags: [print-and-mail, x402, mcp, usdc, lob, postal-mail]
---

# PostAgent — Print & Mail

PostAgent turns a digital document into a **physical letter** that is printed and
mailed to a US address. You upload a document, lock a price quote, and pay per
send with the [x402](https://www.x402.org/) payment protocol (USDC on Base
mainnet). PostAgent then hands the job to [Lob](https://www.lob.com/) for
printing and USPS delivery.

- **Service base URL:** `https://postagent-api.interpretai.tech`
- **MCP endpoint:** `https://postagent-api.interpretai.tech/mcp`

## When to use this skill

Use it when the user wants to **send real physical mail in the US**, e.g.:

- "Mail this PDF letter to 500 Market St, San Francisco."
- "Send a printed notice to these 10 customers."
- "Post this invoice to my client."

Do **not** use it for email, faxes, international mail, or anything that is not a
physical US-to-US letter.

## CRITICAL — read before sending

- **Real money + irreversible.** Submitting a paid job charges USDC and prints &
  mails a physical letter. It cannot be undone. **Always** show the user the
  recipient, sender, page count, and price, and get **explicit confirmation**
  before paying.
- **US → US only.** Both sender and recipient must be US addresses. State must be
  a 2-letter code (e.g. `CA`); ZIP must be 5-digit or ZIP+4.
- **Non-custodial payment.** PostAgent never holds a wallet. The agent's own
  x402-capable wallet must sign and pay the quote. If no wallet is available,
  stop and tell the user — do not attempt to fabricate payment.
- **One letter = one quote + one payment + one job.** There is no batch send. To
  mail N recipients, repeat the quote→pay→job cycle N times (you can reuse the
  same `documentId`).

## How to call PostAgent

Drive the **REST/HTTP API** directly — this works in any agent with shell or
HTTP access and needs no setup, so it is the default path used throughout this
skill. If the PostAgent **MCP server** happens to be configured in your client,
you may use its structured tools instead (they map 1:1 to these steps); see
[Optional: MCP server](#optional-mcp-server) at the end. Either way, payment is
the same non-custodial x402 step (the API never signs for you).

## Workflow

### 1. Upload the document (free)

A document is either a **finished letter** (the stored PDF is mailed as-is) or a
**mail-merge template** (`{{field}}` placeholders filled per recipient).

REST — finished letter from inline Markdown:

```bash
curl -sX POST https://postagent-api.interpretai.tech/v1/documents \
  -H "content-type: application/json" \
  -d '{ "format": "markdown", "content": "# Notice\n\nPlease review the attached terms.", "template": false }'
```

REST — finished letter from an existing PDF (base64 or URL):

```bash
curl -sX POST https://postagent-api.interpretai.tech/v1/documents \
  -H "content-type: application/json" \
  -d '{ "format": "pdf", "url": "https://example.com/letter.pdf", "template": false }'
```

REST — reusable template (must be text-based and contain at least one `{{field}}`):

```bash
curl -sX POST https://postagent-api.interpretai.tech/v1/documents \
  -H "content-type: application/json" \
  -d '{ "format": "html", "content": "<p>Dear {{name}}, your balance is {{amount}}.</p>", "template": true }'
```

Accepted formats: `pdf`, `html`, `markdown`, `text`, `docx`, `image` (PNG/JPEG).
Provide content exactly one way: inline `content` (text formats), `contentBase64`
(binary), or `url`. The response returns a `documentId` (e.g. `doc_...`), the
page count, and — for templates — the detected `mergeFields`.

> **Address zone:** PostAgent automatically reserves the top ~3 inches of page 1
> for the recipient address block, so you do **not** need to leave the top blank.
> This can make the stored/printed page count one greater than your source
> document (e.g. a 1-page PDF becomes 2 pages), which is reflected in the price.

### 2. Get a price quote (free)

Verifies both addresses and locks a USDC price for 15 minutes.

```bash
curl -sX POST https://postagent-api.interpretai.tech/v1/quotes \
  -H "content-type: application/json" \
  -d '{
    "documentId": "doc_XXX",
    "from": { "name": "Sender Name", "line1": "123 Main St", "city": "San Francisco", "state": "CA", "zip": "94105" },
    "to":   { "name": "Recipient Name", "line1": "500 Market St", "city": "San Francisco", "state": "CA", "zip": "94105" },
    "options": { "color": false, "doubleSided": true, "mailClass": "usps_first_class", "certified": false }
  }'
```

For a **template** document, also pass `mergeVariables` with a value for every
field in the document's `mergeFields`, e.g.
`"mergeVariables": { "name": "Jane", "amount": "$42.00" }`.

The response includes:

- `id` — the `quoteId` (e.g. `qt_...`).
- `price.usd`, `price.usdcAtomic` — the locked price (atomic USDC = the exact charge).
- `payment` — `{ asset, network, payTo, amountAtomic }`.
- `paymentUrl` — `…/v1/quotes/{quoteId}/pay`, the canonical x402-payable URL.
- `previewUrl` — `GET` it for a signed PDF preview of the final piece (address
  block + reserved zones drawn on top). Share this with the user when available.
- `expiresAt` — pay before this or re-quote.

**Show the user the recipient, sender, options, page count, price, and (if
available) the preview, then wait for explicit confirmation.**

### 3. Pay the quote → create the letter (charges money, irreversible)

The `paymentUrl` is a standard x402 resource: an unpaid `GET`/`POST` returns
`402 PAYMENT-REQUIRED`; the same request retried with a signed `X-PAYMENT`
header verifies, prints, and settles.

Easiest path — pay in-band with any compliant x402 wallet (e.g. Coinbase's `awal`):

```bash
npx awal@latest x402 pay https://postagent-api.interpretai.tech/v1/quotes/qt_XXX/pay --max-amount 200000
```

(`--max-amount` is in atomic USDC; set it to the quote's `amountAtomic` or higher.)

> If your wallet emits a **detached** signature instead of paying in-band, see
> the MCP `submit_paid_mail_job` flow in
> [Optional: MCP server](#optional-mcp-server).

A successful payment returns a **job** (`job_...`) with `status`,
`providerLetterId`, settlement `payment.transaction`, and `tracking`.

> Optionally append `?webhookUrl=https://you.example.com/hook` to the pay URL to
> receive job status updates.

### 4. Track the job (free)

```bash
curl -s https://postagent-api.interpretai.tech/v1/jobs/job_XXX
```

Returns normalized `status`, `providerLetterId`, `tracking` (incl. `proofUrl`
and thumbnails once Lob renders them), and `failureReason` if any.

## Pricing

Deterministic and transparent:

```
total = ceil((base + per_page*pages + color? + mail_class? + certified?) * margin)
```

The quote returns the total in USD and as atomic USDC (6 decimals); the atomic
amount is the exact x402 charge. Color, certified mail, and first-class each add
a surcharge.

## Options reference

| Option | Values | Default |
| ------ | ------ | ------- |
| `color` | `true` / `false` | `false` |
| `doubleSided` | `true` / `false` | `true` |
| `mailClass` | `usps_first_class` / `usps_standard` | `usps_first_class` |
| `certified` | `true` / `false` | `false` |

## Error handling

- `402` on the pay URL **without** a payment header is expected — it is the
  payment challenge, not a failure.
- Undeliverable address → `422` with a `deliverability` reason. Fix the address
  and re-quote.
- Expired quote → re-run step 2; quotes live 15 minutes.
- Quotes/jobs are immutable once created; never retry a settled payment.

## Optional: MCP server

If your client has the PostAgent [MCP](https://modelcontextprotocol.io) server
configured, you can use its structured tools instead of raw HTTP. This is an
enhancement, not a requirement — the REST workflow above is fully sufficient,
and a skill cannot register the MCP server for you. Add it to your client config:

```json
{
  "mcpServers": {
    "PostAgent": {
      "url": "https://postagent-api.interpretai.tech/mcp"
    }
  }
}
```

The tools map 1:1 to the workflow steps:

| Tool | Workflow step | Charges? |
| ---- | ------------- | -------- |
| `create_letter` | Upload a finished document (identical for all recipients). `{{...}}` is printed literally. | No |
| `create_template` | Upload an html/markdown/text template with `{{fields}}` for mail merge. | No |
| `create_mail_quote` | Verify addresses + lock a 15-minute USDC price. | No |
| `prepare_mail_payment` | Fetch the x402 `PAYMENT-REQUIRED` challenge for a quote. | No |
| `submit_paid_mail_job` | Settle a **detached** x402 signature and create the letter. **Irreversible.** | **Yes** |
| `get_mail_job_status` | Look up status / tracking for a job. | No |

`submit_paid_mail_job` is the detached-signature alternative to paying the
`paymentUrl` in-band; it requires both a signed x402 `paymentSignature` and
`userConfirmed: true`. Set `userConfirmed: true` only after the human explicitly
approved the recipient, sender, content, and price. Read-only resources are also
exposed: `postagent://terms`, `postagent://privacy`, `postagent://formats`,
`postagent://pricing`.

## Reference

- Terms / Privacy: `GET https://postagent-api.interpretai.tech/v1/terms`
- x402 protocol: https://www.x402.org/
- MCP server (optional): `https://postagent-api.interpretai.tech/mcp`
