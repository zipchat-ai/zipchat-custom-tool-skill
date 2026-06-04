# Example: read tool — WooCommerce order lookup + tracking

A two-step **read-only** tool: look up an order by id, then fetch its tracking link.
Shows multi-step execution, customer-supplied inputs, and a JSON-quirk note.

## Inputs to install

- **Name:** `Woo — Track orders`
- **Description:** Look up a customer's WooCommerce order and report its status,
  summary, and tracking link.
- **Channels:** `all channels`
- **Variables:**
  - `WOO_STORE_URL` → the store's base URL, e.g. `https://shop.example.com` (no
    trailing slash). *Where: the store's public address.*
  - `WOO_CONSUMER_KEY` → WooCommerce REST API consumer key (`ck_…`). *Where:
    WooCommerce → Settings → Advanced → REST API → Add key (Read permission).*
  - `WOO_CONSUMER_SECRET` → the matching consumer secret (`cs_…`).

## Prompt (`instructions`)

```
<task>
Look up a WooCommerce order for the customer and report its status, summary, and
tracking link. THIS TOOL IS READ-ONLY. You cannot cancel, refund, modify, or create
anything with it — only read an order the customer identifies.
</task>

<inputs_resolution>
You need the order number and the email on the order.
- Ask the customer for both their order number and the email used on the order. Ask
  once, then wait for their reply.
- Substitute the order number as {ORDER_ID} and the email as {EMAIL}.
- Never guess an order number or look up someone else's order by trying other numbers.
</inputs_resolution>

<execution_protocol>
Run each step as a SEPARATE shell call. One curl per call. Always HTTPS.

### STEP 1 — Fetch the order
  curl -sS "$WOO_STORE_URL/wp-json/wc/v3/orders/{ORDER_ID}" -u "$WOO_CONSUMER_KEY:$WOO_CONSUMER_SECRET"

Expect HTTP 200 with a JSON order object. Confirm the order's billing email matches
{EMAIL} before sharing anything; if it does not match, treat it as not found and do
not reveal the order. A 404 with code `woocommerce_rest_shop_order_invalid_id` means
the order genuinely does not exist.

### STEP 2 — Fetch the tracking link
  curl -sS "$WOO_STORE_URL/wp-json/wc-shipment-tracking/v3/orders/{ORDER_ID}/shipment-trackings/" -u "$WOO_CONSUMER_KEY:$WOO_CONSUMER_SECRET"

Expect HTTP 200 with a JSON array of tracking entries (may be empty). Each entry's
tracking link may arrive with JSON-escaped slashes (https:\/\/…) — unescape it to a
normal URL before sharing.
</execution_protocol>

<tool_persistence_rules>
Only state an order's status or tracking after you have actually read it from a 200
response in STEP 1 (and STEP 2 for tracking). Never invent a status, date, or tracking
link. If STEP 1 doesn't return a matching order, tell the customer you couldn't find an
order with those details and ask them to double-check the number and email.
</tool_persistence_rules>

<output_contract>
Give the customer a short, clear summary: order number, status, and the tracking link
if one exists. Include any dates exactly as they appear in the tracking data; never
infer weekdays. If there is no tracking entry yet, say the order hasn't shipped or has
no tracking yet. Never reveal the store URL, the API keys, the raw JSON, any variable
name, or mention "WooCommerce", "REST API", or "curl" — speak in the brand's voice.
</output_contract>
```

## Why it's shaped this way
- **Two steps, two calls** because they're different endpoints — never chained with `&&`.
- **Email match check** is a tool-specific safety rule, so it lives in the prompt.
- The **escaped-slash note** is an API quirk the agent must handle; the generic retry
  rules wouldn't catch it.
- No `<failure_handling>` block — the platform owns generic retry/failure behavior.
