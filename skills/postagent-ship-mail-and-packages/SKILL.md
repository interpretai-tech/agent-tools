---
name: postagent-ship-mail-and-packages
description: Print and physically mail documents and postcards to US postal addresses via the PostAgent API, send certified/registered mail with proof of delivery, verify postal addresses for deliverability, and (KYC-verified senders only) run bulk mail campaigns. Use when the user wants to send real, physical mail — a letter, notice, invoice, or postcard — to a US recipient, or to check whether a postal address is deliverable. Each send is paid per-call in USDC on Base using the x402 protocol, so it spends real money and is irreversible once submitted.
license: MIT
version: 0.2.0
metadata:
  homepage: https://postagent-api.interpretai.tech
  tags: [print-and-mail, x402, mcp, usdc, lob, postal-mail]
---

# PostAgent — Print & Mail

PostAgent turns a digital document into **physical mail** that is printed and
delivered via USPS. You upload a document, lock a price quote, and pay per send
with the [x402](https://www.x402.org/) payment protocol (USDC on Base mainnet).
PostAgent then hands the job to [Lob](https://www.lob.com/) for printing and
USPS delivery.

Products:

- **Letters** — PDF/HTML/Markdown/text/DOCX/image, optional color, certified /
  certified-with-return-receipt / registered mail.
- **Postcards** — 4x6 / 6x9 / 6x11, full color, from front+back artwork
  (4x6 cards can ship internationally).
- **Address verification** — standalone paid deliverability check (US CASS or
  international), ~2¢ per call.
- **Bulk campaigns** — one creative, up to 500 recipients, via the provider's
  campaign pipeline. **KYC-walled: unavailable until identity verification
  launches** (see [Bulk campaigns](#bulk-campaigns-kyc-required)).

- **Service base URL:** `https://postagent-api.interpretai.tech`
- **MCP endpoint:** `https://postagent-api.interpretai.tech/mcp`

## When to use this skill

Use it when the user wants to **send real physical mail in the US**, e.g.:

- "Mail this PDF letter to 500 Market St, San Francisco."
- "Send a printed notice to these 10 customers."
- "Post this invoice to my client." / "Send it certified with return receipt."
- "Mail a postcard to…" / "Is this address deliverable?"

Do **not** use it for email, faxes, or packages/parcels. International delivery
is supported **only** for 4x6 postcards; all letters are US-to-US.

## CRITICAL — read before sending

- **Real money + irreversible.** Submitting a paid job charges USDC and prints &
  mails a physical piece. It cannot be undone. **Always** show the user the
  recipient, sender, page count, and price, and get **explicit confirmation**
  before paying.
- **US senders + US recipients** for letters. State must be a 2-letter code
  (e.g. `CA`); ZIP must be 5-digit or ZIP+4. Exception: a **4x6 postcard** may
  go to an international recipient (set `to.country` to the 2-letter ISO code);
  the sender must still be a US address.
- **Non-custodial payment.** PostAgent never holds a wallet. The agent's own
  x402-capable wallet (or a Shared Payment Token it mints) must sign and pay the
  quote — never fabricate payment. If you have no payment method yet, don't just
  fall back to hosted checkout: pick a rail and set one up (see
  [Payment methods & setup](#payment-methods--setup) below).
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

> **Images / pictures.** To mail a picture *as the letter*, upload it with
> `format: "image"` (PNG/JPEG via `contentBase64` or `url`). To place a picture
> *inline* in an `html`/`markdown` letter, it MUST be embedded as a base64 data
> URI (e.g. `<img src="data:image/png;base64,iVBORw0KGgo...">` or
> `![](data:image/png;base64,...)`). External image URLs such as
> `<img src="https://...">` are **blocked and render blank** — the PDF renderer
> runs with no network access for security, so only `data:` images load.
> Already have a composed image-bearing document? Send it as `format: "pdf"`.

Easiest way to mail a standalone picture — pass a `url` and the **server**
fetches it (no base64 needed):

```bash
curl -sX POST https://postagent-api.interpretai.tech/v1/documents \
  -H "content-type: application/json" \
  -d '{ "format": "image", "url": "https://example.com/photo.png", "template": false }'
```

To inline a **local** image inside an html/markdown letter, encode it to a
base64 data URI and send the request body **from a file** — a real image's
base64 is far too large to place on the command line:

```bash
# 1. Encode the image into a data URI (one long line, no newlines).
IMG="data:image/png;base64,$(base64 -i logo.png | tr -d '\n')"
# 2. Write the JSON body to a file so the big blob never hits the shell args.
cat > body.json <<JSON
{ "format": "html", "template": false,
  "content": "<h1>Hello</h1><img src=\"$IMG\" style=\"max-width:400px\" />" }
JSON
# 3. POST the file (note --data-binary @file, not -d).
curl -sX POST https://postagent-api.interpretai.tech/v1/documents \
  -H "content-type: application/json" --data-binary @body.json
```

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
    "options": { "color": false, "doubleSided": true, "mailClass": "usps_first_class" }
  }'
```

Need **proof of mailing/delivery**? Add `"extraService"` to `options`:
`"certified"` (USPS Certified Mail), `"certified_return_receipt"` (certified +
electronic return receipt — proof of delivery), or `"registered"` (maximum
chain-of-custody for valuables). All three require `usps_first_class` and add a
surcharge that appears in the quoted price. (`"certified": true` still works as
a legacy alias for `"extraService": "certified"`.)

For a **template** document, also pass `mergeVariables` with a value for every
field in the document's `mergeFields`, e.g.
`"mergeVariables": { "name": "Jane", "amount": "$42.00" }`.

The response includes:

- `id` — the `quoteId` (e.g. `qt_...`).
- `price.usd`, `price.usdcAtomic` — the locked price (atomic USDC = the exact charge).
- `payment` — `{ asset, network, payTo, amountAtomic }`.
- `paymentOptions` — ranked rails the agent can choose from (preferred first):
  `x402` (autonomous, USDC on Base), `mpp` (autonomous fiat card/wallet via a
  Shared Payment Token, when enabled), and `stripe_checkout` (a hosted credit-
  card page for a human payer, the last resort).
- `paymentUrl` — `…/v1/quotes/{quoteId}/pay`, the canonical x402-payable URL.
- `previewUrl` — `GET` it for a signed PDF preview of the final piece (address
  block + reserved zones drawn on top). Share this with the user when available.
  Programmatic clients (Accept: application/json) get JSON with the signed `url`;
  opening it in a browser redirects straight to the PDF.
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

**Paying by credit card (human fallback).** If no agent wallet is available and
the payer is a human, `POST /v1/quotes/{quoteId}/checkout` (or the MCP
`create_card_checkout` tool) returns a hosted Stripe `checkoutUrl`. Hand it to
the user to open and pay; the letter is created automatically once payment
completes (poll job status). When MPP is enabled, an agent holding a Shared
Payment Token can instead `POST /v1/quotes/{quoteId}/pay/mpp` with
`{ "sharedPaymentToken": "spt_…" }` to pay autonomously by card/wallet (fiat).

## Payment methods & setup

Each quote returns a ranked `paymentOptions` array. Try them in order and use the
first you can satisfy. **If you have no payment method yet, don't dead-end at
checkout — set one up:**

- **x402 — USDC on Base (autonomous, preferred).** Pay the quote's `paymentUrl`
  in-band with any compliant x402 wallet:
  `npx awal@latest x402 pay <paymentUrl> --max-amount <amountAtomic>`.
  *No wallet?* Install one (`npx awal@latest`), connect it, and fund it with USDC
  on Base mainnet (at least the quote's `price.usdcAtomic` plus gas), then pay.
- **MPP — fiat card/wallet via a Stripe Shared Payment Token (autonomous).** Mint
  an SPT scoped to the seller, then `POST /v1/quotes/{quoteId}/pay/mpp` with
  `{ "sharedPaymentToken": "spt_…" }` (or MCP `pay_mail_with_shared_payment_token`).
  The seller profile to scope to is on the quote: `paymentOptions[mpp].details.stripeProfileId`.
  *No payment method?* Use Stripe's Link CLI (see `link.com/agents`):
  `npx @stripe/link-cli auth login` → `payment-methods list` →
  `spend-request create --payment-method-id <pm> --amount <cents> --credential-type shared_payment_token --network-id <stripeProfileId> --request-approval`,
  then pass the returned `spt_…`. (US-only; 0.50 USD card minimum.)
- **Hosted Checkout — human with a card (last resort).** `POST /v1/quotes/{quoteId}/checkout`
  (or MCP `create_card_checkout`) returns a `checkoutUrl` for a human to pay in a
  browser; the letter is created once payment completes (poll job status).

> MCP clients can read the `postagent://payment-methods` resource for this same
> guidance, with per-rail examples and setup steps, in structured form.

### 4. Track the job (free)

```bash
curl -s https://postagent-api.interpretai.tech/v1/jobs/job_XXX
```

Returns normalized `status`, `providerLetterId`, `tracking` (incl. `proofUrl`
and thumbnails once Lob renders them), and `failureReason` if any.

## Postcards

Postcards are built from **two artwork files** (front + back), each a
single-page PDF, PNG, or JPEG stored **verbatim** — no page normalization, no
reserved address zone. You are responsible for trim size + bleed: 4x6 needs
4.25"x6.25" artwork, 6x9 needs 6.25"x9.25", 6x11 needs 6.25"x11.25" (0.125"
bleed each edge). Lob prints the recipient address block over part of the
**back**, so keep that area clear. Postcards always print in full color.

```bash
# 1. Upload front and back separately with usage: "postcard"
curl -sX POST https://postagent-api.interpretai.tech/v1/documents \
  -H "content-type: application/json" \
  -d '{ "usage": "postcard", "format": "image", "url": "https://example.com/front.png" }'
# → { "id": "doc_FRONT", "kind": "postcard_art", ... }   (repeat for the back)

# 2. Quote with product: "postcard"
curl -sX POST https://postagent-api.interpretai.tech/v1/quotes \
  -H "content-type: application/json" \
  -d '{
    "product": "postcard",
    "frontDocumentId": "doc_FRONT",
    "backDocumentId": "doc_BACK",
    "from": { "name": "Sender", "line1": "123 Main St", "city": "San Francisco", "state": "CA", "zip": "94105" },
    "to":   { "name": "Recipient", "line1": "500 Market St", "city": "San Francisco", "state": "CA", "zip": "94105" },
    "options": { "size": "4x6", "mailClass": "usps_first_class" }
  }'
```

Pay the returned `paymentUrl` exactly like a letter (step 3 above). Sizes:
`4x6` (default), `6x9`, `6x11`. **International:** only `4x6`, only
`usps_first_class` — give `to.country` the 2-letter ISO code and use the
looser address shape (free-form `state`, postal code in `zip`). The quote's
`previewUrl` returns signed URLs of the raw front/back artwork; Lob's rendered
proof appears on the job's `tracking` after payment.

## Address verification (paid, ~2¢)

Standalone deliverability check — no mail is sent. US addresses get
CASS-standardized results (ZIP+4); other countries run international
verification. The endpoint is a stable x402 resource: unpaid requests return
the `402` challenge, and the same `POST` retried with a payment header returns
the result synchronously.

```bash
npx awal@latest x402 pay https://postagent-api.interpretai.tech/v1/verify \
  --max-amount 20000 \
  -X POST -d '{ "address": { "line1": "500 Market St", "city": "San Francisco", "state": "CA", "zip": "94105" } }'
# → { "deliverable": true, "deliverability": "deliverable", "normalized": { ... } }
```

International: add `"country": "GB"` (2-letter ISO) and put the postal code in
`zip` or `postalCode`. Note: when mailing through PostAgent you do **not** need
this — quotes already verify both addresses for free.

## Bulk campaigns (KYC required)

`product: "campaign"` on `POST /v1/quotes` quotes ONE bulk send to up to 500
recipients: a single template (per-recipient `{{merge_variables}}`) or static
PDF, fulfilled through Lob's campaign pipeline (campaign + creative + audience
upload). Price = per-letter unit price x recipient count. Payment is **x402
only**, and the request must name the paying wallet (`payerWallet`).

> **KYC WALL — currently unusable.** Campaigns require the paying wallet to be
> identity-verified with PostAgent. KYC onboarding is **not yet available**, so
> every campaign quote currently returns `403 kyc_required`. **Do not retry.**
> To send bulk mail today, run the normal letter quote→pay cycle once per
> recipient (same `documentId`, per-recipient `mergeVariables`).

Once KYC launches: after payment, the provider validates every recipient
address asynchronously; poll `GET /v1/quotes/{quoteId}/job` (which gains a
`campaign` object), then `GET /v1/campaigns/{campaignId}` for
`validating → sent | partial | failed`, counts, and a row-by-row `failuresUrl`.
Failed recipients are excluded from the send and their share of the price is
refunded (processed manually, not automatic). Set `options.useType` honestly
(`marketing` is the default for bulk sends).

## Pricing

Deterministic and transparent:

```
letter   total = ceil((base + per_page*pages + color? + mail_class? + extra_service?) * margin)
postcard total = ceil((size_base + mail_class?) * margin)        # always full color
campaign total = letter_unit_price * recipient_count
verify   total = flat fee (~2¢)
```

The quote returns the total in USD and as atomic USDC (6 decimals); the atomic
amount is the exact x402 charge. Color, first-class, and each extra service
(certified, certified + return receipt, registered) add a surcharge.

## Options reference

Letters (`options` on a default/letter quote):

| Option | Values | Default |
| ------ | ------ | ------- |
| `color` | `true` / `false` | `false` |
| `doubleSided` | `true` / `false` | `true` |
| `mailClass` | `usps_first_class` / `usps_standard` | `usps_first_class` |
| `extraService` | `certified` / `certified_return_receipt` / `registered` | none |
| `certified` | `true` / `false` (legacy alias for `extraService: certified`) | `false` |

Postcards (`options` with `product: "postcard"`):

| Option | Values | Default |
| ------ | ------ | ------- |
| `size` | `4x6` / `6x9` / `6x11` | `4x6` |
| `mailClass` | `usps_first_class` / `usps_standard` | `usps_first_class` |

Campaigns add `useType` (`marketing` / `operational`, default `marketing`) on
top of the letter options (no `certified` alias; use `extraService`).

## Error handling

- `402` on the pay URL **without** a payment header is expected — it is the
  payment challenge, not a failure.
- Undeliverable address → `422` with a `deliverability` reason. Fix the address
  and re-quote.
- Expired quote → re-run step 2; quotes live 15 minutes.
- Quotes/jobs are immutable once created; never retry a settled payment.
- If calls fail in a confusing way (connection errors, unexpected `5xx`),
  sanity-check connectivity and your base URL with
  `curl https://postagent-api.interpretai.tech/health` — a healthy service
  returns `{"status":"ok","service":"PostAgent"}`. This is optional and only for
  troubleshooting; do not gate the workflow on it.

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
| `create_postcard_art` | Upload one side of a postcard (single-page pdf/png/jpeg, stored verbatim). | No |
| `create_mail_quote` | Verify addresses + lock a 15-minute USDC price for a letter. | No |
| `create_postcard_quote` | Lock a price for a postcard (front + back artwork, size, recipient). | No |
| `create_campaign_quote` | Lock a price for a bulk campaign (1-500 recipients). **Currently returns `kyc_required` — KYC onboarding not yet available.** | No |
| `verify_address` | Standalone paid deliverability check (US or international), flat ~2¢ x402 fee. | **Yes (~2¢)** |
| `prepare_mail_payment` | Fetch the x402 `PAYMENT-REQUIRED` challenge for a quote. | No |
| `submit_paid_mail_job` | Settle a **detached** x402 signature and create the letter/postcard. **Irreversible.** | **Yes** |
| `pay_mail_with_shared_payment_token` | Pay a quote autonomously by card/wallet (fiat) with a Stripe Shared Payment Token (MPP); creates the piece. **Irreversible.** | **Yes** |
| `create_card_checkout` | Create a hosted Stripe card-checkout link for a human payer (last resort). | No (human pays later) |
| `get_mail_job_status` | Look up status / tracking for a job. | No |
| `get_campaign_status` | Look up a campaign's validation/sending progress and failure report. | No |

`submit_paid_mail_job` is the detached-signature alternative to paying the
`paymentUrl` in-band; it requires both a signed x402 `paymentSignature` and
`userConfirmed: true`. Set `userConfirmed: true` only after the human explicitly
approved the recipient, sender, content, and price. Read-only resources are also
exposed: `postagent://terms`, `postagent://privacy`, `postagent://formats`,
`postagent://pricing`, and `postagent://payment-methods` (every supported rail
with examples + step-by-step setup for a payer that has no wallet/token yet).

## Reference

- Terms / Privacy: `GET https://postagent-api.interpretai.tech/v1/terms`
- x402 protocol: https://www.x402.org/
- MCP server (optional): `https://postagent-api.interpretai.tech/mcp`
