---
name: zipchat-custom-tool
description: >
  Build a Zipchat (chatlive) custom tool — the prompt/instructions, variables, and
  (for gallery templates) the merchant note — that lets the AI support agent carry out
  a real action by running shell commands (curl plus jq, ripgrep, fd, unzip) against one
  or more external HTTP APIs. Trigger whenever someone wants to "create/build/draft a
  custom tool", "connect Zipchat to <API>", "make the bot look up orders / create
  discounts / fetch tracking / escalate tickets", write the instructions/prompt for a
  custom tool, or debug/fix a custom tool that isn't working (it can read the debug
  transcript from the dashboard's Debug modal). Produces all the inputs needed to
  install the tool; the author does NOT need access to the chatlive codebase.
---

# Building a Zipchat custom tool

A **custom tool** lets a brand's Zipchat AI agent perform a real action by running shell
commands in a sandbox against one or more external HTTP APIs. It can be a single call, a
sequence of calls against the same API, or a multi-system flow that opens several APIs
one after another — order lookup, discount creation, shipment tracking, ticket
escalation, and richer multi-step workflows are all in scope. Your job with this skill is
to interview the user and produce **every input needed to install the tool**, with a
well-formed prompt as the centerpiece.

You do not need the chatlive codebase. Everything you need to know about how tools
run is below.

## How a custom tool actually executes (mental model)

1. The brand installs a tool with: a **name**, a **description**, a **prompt**
   (called `instructions`), and a set of **variables** (secrets/config).
2. At conversation time, Zipchat injects the tool's prompt into the agent's system
   prompt, wrapped as:
   ```
   #### <tool name>
   <custom_tool_instructions>
   …your prompt…
   </custom_tool_instructions>
   ```
3. When a customer's request matches the tool, the agent executes the shell command(s)
   the prompt specifies — typically `curl`, optionally piped through `jq` to pull or
   reshape fields, and across multiple steps when the tool needs them. Each command's
   output (usually JSON) comes back to the agent, which reads it and either runs the next
   step or replies to the customer in the brand's voice.
4. The tool's **variables** are injected into the sandbox as **environment variables**.
   So a variable named `WOO_CONSUMER_KEY` is referenced in the curl as
   `$WOO_CONSUMER_KEY`. Values are encrypted at rest and never shown to the customer.

There is no server-side code per tool — the effect is produced by the real HTTP call(s)
the prompt makes. If a desired action has no HTTP call behind it, the tool can't do it.

### What the sandbox has
The shell is a real Unix shell preloaded with a small toolbox — use it to *process* what
a call returns:

- `curl` — every HTTP call (the only thing that produces an effect).
- `jq` — parse, filter, and reshape JSON; e.g. pipe a response through `jq -r '.id'` to
  feed the next step.
- `ripgrep` (`rg`) and `fd` — search text/files the tool fetched.
- `unzip` — expand an archive a call returns.

The actual effect must always come from the HTTP call itself. Never use the shell to fake
a result, stand in for an API call, or mutate sandbox files to simulate an effect.

## Two kinds of values: `$VAR` vs `{PLACEHOLDER}`

This distinction is the heart of a good tool prompt.

| Form | Meaning | Example | Who fills it |
|---|---|---|---|
| `$VAR_NAME` | A stored **variable** (secret/config) injected as an env var | `$WOO_STORE_URL`, `$ZENDESK_API_TOKEN` | The platform, at runtime |
| `{PLACEHOLDER}` | A **runtime value** the agent substitutes from the conversation | `{ORDER_ID}`, `{EMAIL}`, `{PERCENT}` | The agent, when writing the curl |

- Put anything secret or brand-specific (API keys, store URL, subdomain, caps) in a
  **variable** and reference it as `$VAR`. Never hardcode a secret in the prompt.
- Use `{PLACEHOLDER}` for values that come from the customer or the agent's judgment.
  Tell the agent in `<inputs_resolution>` how to obtain each one.

### Conversation context: `{CONVERSATION_ID}` and `{CHAT_ID}` are always in the prompt

Every tool-enabled agent is told the **current conversation id and chat id** in its
system prompt (the shared Custom Tool Mode section). When a tool's API call needs to
identify the conversation it runs on — escalate it, attach a note, tag or log it, look it
up — reference these as the `{CONVERSATION_ID}` and `{CHAT_ID}` placeholders and, in
`<inputs_resolution>`, tell the agent to read the exact values from its system prompt.
Don't store them as `$VAR`s (they change every conversation) and never ask the customer
for them.

Example — escalate the current Zipchat conversation via the backend API:
```
curl -sS -X PATCH "$ZIPCHAT_BASE_URL/chats/{CHAT_ID}/conversations/{CONVERSATION_ID}/escalation" -H "Authorization: Bearer $ZIPCHAT_API_KEY"
```

### Variable naming rules (enforced)
- Uppercase letters, digits, underscores only; cannot start with a digit. Regex:
  `^[A-Z_][A-Z0-9_]*$`.
- **Must be globally unique across all of a brand's active tools** — so always prefix:
  `WOO_…`, `ZENDESK_…`, `LOYALTY_…`. Never a bare `API_KEY`.
- Variables live in a **per-chat pool shared across that chat's tools**: define one once
  and the same `$VAR` can be reused by another tool in the same chat (one stored value,
  edited in one place). This is exactly why names must be unique per brand and prefixed —
  a collision would mean two tools fighting over one value.

### Shopify stores: use the managed `$SHOPIFY_API_KEY` (don't ask for a token)

When the tool runs in a **Shopify** store's chat, the platform provides a built-in
**managed variable, `$SHOPIFY_API_KEY`**, that resolves at runtime to the store's live
Shopify Admin API access token. It's auto-provisioned for every Shopify chat, can't be
deleted, and attaches to a tool like any other variable.

What this means when you author a Shopify tool:

- **Use `$SHOPIFY_API_KEY` as the Admin API token** — pass it in the
  `X-Shopify-Access-Token` header. Do **not** create your own variable for the token, and
  do **not** write a merchant note telling them to create a custom app or paste an Admin
  API token. That whole setup step is gone.
- **`SHOPIFY_API_KEY` is a reserved name.** You can't define a user variable called
  `SHOPIFY_API_KEY` — the install form rejects it. Reference the managed one instead.
- **Only the token is managed.** You still need the store's `*.myshopify.com` admin
  domain for the URL — supply it as a stored variable (e.g. `SHOPIFY_SHOP_DOMAIN`) or a
  `{PLACEHOLDER}` resolved from the store context; it is not part of the managed key.
- **Scopes are gated by the managed key's Permissions.** The token only carries the
  scopes the merchant has granted. A baseline set is always present (today:
  `read_products`, `read_orders`, `read_all_orders`, `read_content`, `read_themes`,
  `read_inventory`, `write_customers`, `write_discounts`, plus pixel/event scopes).
  Anything beyond it — e.g. `write_orders`, `write_products`, `read_draft_orders`,
  `read_fulfillments`/`write_fulfillments`, `read_price_rules`, `read_locations` — must be
  enabled on the managed key's **Permissions** screen, which re-auths the store through
  Zipchat's official Shopify app. If your tool needs a scope beyond the baseline, **call
  out the exact scope(s)** so the merchant grants them first; otherwise the call returns
  403 / unauthorized.

Example call (Admin REST, API version `2026-04`):
```
curl -sS "https://$SHOPIFY_SHOP_DOMAIN/admin/api/2026-04/orders/{ORDER_ID}.json" -H "X-Shopify-Access-Token: $SHOPIFY_API_KEY"
```

Non-Shopify tools (WooCommerce, Zendesk, a custom backend, …) are unaffected — keep using
stored `$VAR`s for their credentials as before.

## What platform rules ALREADY cover — do NOT repeat these in your prompt

Zipchat injects shared "Custom Tool Mode" rules into every tool-enabled agent. Your
prompt must **not** restate them (it wastes tokens and drifts out of sync):

- **Command hygiene** — keep each command on one line, the URL unbroken, quotes
  balanced. Piping a response through `jq`/`awk`/etc. to post-process it is fine; running
  the real command (not an `echo`/`python -c` that fakes a result) is required.
- **Fix-and-retry** — if a request is malformed or rejected before causing any effect
  (syntax error, empty/non-JSON body, HTTP 400/422, `rest_no_route`, bad path/param),
  the agent re-reads, fixes, and retries up to 3 times. Safe even for write tools.
- **Don't blindly retry** — a genuine "not found" / 401 / 403 won't change on retry;
  and a write tool that hit an ambiguous outcome (timeout-after-send, 5xx) may have
  partly succeeded, so it isn't blindly re-fired.
- **No fishing** — on a genuine not-found/rejection, the agent won't probe alternate
  IDs/params.
- **Honest failure** — when the tool can't complete, the agent reports briefly in the
  brand's voice and never leaks error bodies, status codes, credentials, or URLs.
- **Secret safety** — env var *values* are never revealed; files in the sandbox are
  never modified.

So your prompt should focus on **what's specific to this tool**: which endpoint, how
to resolve inputs, business rules, and how to present the result. Only add error
handling that is *tool-specific* (e.g. "if the code already exists, generate a new one
before retrying") — never the generic loop.

## The inputs you must produce

When you finish, hand the user a filled-in version of all of these:

1. **Name** — short, human-readable, ≤255 chars (e.g. "Woo — Track orders"). It's
   shown to the agent as the section header.
2. **Description** — one line on what the tool does / when to use it (gallery/UX copy).
3. **Variables** — a list of `NAME → what to paste` pairs (the secrets/config), each
   with a note on where the merchant gets the value. These become `$VAR`s.
4. **Prompt (`instructions`)** — the core deliverable, ≤10,000 chars. Structure below.
5. **(Optional) Note** — merchant-facing setup instructions for a gallery template
   (e.g. "How to get your REST API key", "what to enter for the store URL").

## Prompt structure

Write the prompt as Markdown with these XML-ish sections, in order. See
`reference/prompt-structure.md` for the full section-by-section guide, and
`examples/` for two complete, real prompts (a read tool and a write tool).

```
<task>            What the tool does + when to use it. If it writes/creates data, say so loudly.
<inputs_resolution>  Every {PLACEHOLDER} and how the agent obtains it (ask customer, read system prompt, decide).
<execution_protocol> The shell command(s): one network call per step (number multi-step / multi-system flows), plus any jq post-processing, using $VARS and {PLACEHOLDERS}. State the expected success status.
<tool_persistence_rules>  Anti-hallucination: the agent must observe the real success response before claiming the action happened.
<output_contract>    How to present the result to the customer; what to never reveal (secrets, raw JSON, variable names, the API's name).
```

Add `<composition_rules>` or a tool-specific `<…_rules>` block only when the tool has
real business logic (e.g. a discount cap, a multi-step order). Keep it tight.

## Workflow

1. **Interview.** Ask the user for: a link to (or paste of) the API's **documentation**
   and its auth method — work out the correct endpoints, parameters, and call sequence
   from the docs yourself rather than asking them to spell out each call; what the tool
   should do; what inputs come from the customer vs. stored config; any business
   rules/caps; and whether this is a one-off install or a gallery template (template ⇒
   also write a note). If the target API is the merchant's own **Shopify** store, skip
   the auth interview — use the managed `$SHOPIFY_API_KEY` (see the Shopify section above)
   and confirm which scopes the tool needs.
2. **Test the call first if possible.** If you can reach the API, run the real call(s)
   once to confirm the endpoints, auth, and response shapes before writing the prompt.
   A prompt written against a guessed response shape is usually wrong; for a multi-step
   flow, confirm each step in order.
3. **Draft** the prompt + all inputs using the structure above.
4. **Validate** against the checklist below.
5. **Deliver** every input clearly labeled so the user can paste them into the
   install form. Variables and the note go separately from the prompt.
6. **Debug.** If the user reports that the installed tool misbehaves, do not guess —
   ask for the debug transcript and follow "When an installed tool doesn't work" below.

## When an installed tool doesn't work — ask for the debug transcript

If the user says the tool misbehaves (never fires, fails, or the agent replies
wrong), ask them to fetch the **debug transcript** for a real example before you
change anything:

1. In the Zipchat dashboard, open **Conversations** — or reproduce the problem in the
   **Test chat** sidebar — and find the AI reply where the tool should have acted.
2. Click **Debug** under that reply, then switch to the **RAW debug info** tab.
3. Click **Copy debug info** and paste the transcript back into this chat.

The transcript is a plain-text dump of everything the agent saw and did for that one
reply:

- `## System prompt` — the full prompt the agent received. An installed tool appears
  inside it as a `#### <tool name>` section wrapped in `<custom_tool_instructions>`.
- `## User` / `## Assistant` — the conversation turns sent to the model.
- `[Tool call] Shell command` — each shell invocation with its arguments, including
  the exact command line the agent composed (`$VAR` references are shown unexpanded;
  the values never appear).
- `## Tool result` — what came back from each command: its stdout/stderr, i.e. the raw
  API response body or error.

It deliberately contains no model, token, or other platform-internal data — everything
in it is relevant to your tool.

### How to read it

Work through these checks in order; each maps a symptom to the input you should fix:

1. **Is the tool's prompt in the system prompt at all?** No
   `<custom_tool_instructions>` block with your tool's name ⇒ the tool is inactive,
   not installed, or not enabled for that conversation's channel. Fix activation —
   the prompt itself isn't the problem yet.
2. **Did the agent attempt a tool call?** Tool present but no `[Tool call]` where one
   was expected ⇒ the `<task>` trigger description didn't match how the customer
   actually asked. Sharpen the when-to-use wording (add the phrasings customers use).
3. **Is the composed command right?** Compare the curl in the tool call against the
   API docs: endpoint, method, auth header, and `$VAR` names — a typo'd variable name
   expands to an empty string, which typically shows up as a 401 or a malformed URL.
   Check each `{PLACEHOLDER}` was filled with a sensible value from the conversation;
   if not, tighten `<inputs_resolution>`.
4. **What did the API answer?** The tool result's stdout is the ground truth: 401/403
   ⇒ wrong or missing variable value; 404/`rest_no_route` ⇒ wrong path or store URL;
   400/422 ⇒ wrong params; a successful response the agent then misreported ⇒ the
   response shape doesn't match your `jq` filter or `<output_contract>`.
5. **Patch and retest.** Hand the user the corrected input(s) — usually just the
   prompt or one variable — and have them retry the same request in **Test chat**,
   pulling a fresh transcript if it still fails.

## Validation checklist

- [ ] Every secret/brand value is a `$VAR`, never hardcoded; names are prefixed and
      match `^[A-Z_][A-Z0-9_]*$`.
- [ ] Shopify tools use the managed `$SHOPIFY_API_KEY` (no custom-app token variable, no
      "paste a token" note); any scope beyond the baseline is called out for the merchant
      to grant on the managed key's Permissions screen.
- [ ] Every `{PLACEHOLDER}` has a resolution rule in `<inputs_resolution>`.
- [ ] One network call per step (multi-step / multi-system flows are numbered); each
      command stays on one line. Piping through `jq` to post-process is fine; no faking
      results and no mutating sandbox files.
- [ ] HTTPS only; the URL is on one line.
- [ ] The prompt does NOT repeat the platform's generic retry/failure/secret rules.
- [ ] `<tool_persistence_rules>` forbids claiming success before the real success
      response is observed (critical for write tools — discounts, escalations, etc.).
- [ ] `<output_contract>` lists what to never reveal: env var values, raw JSON,
      variable names, and the underlying API/vendor name (speak in the brand's voice).
- [ ] Write tools: `<task>` states it creates/modifies data; any tool-specific retry
      nuance (e.g. regenerate a value on conflict) is inline.
- [ ] Gallery templates include a setup note.
