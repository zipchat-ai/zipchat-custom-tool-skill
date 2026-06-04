---
name: zipchat-custom-tool
description: >
  Build a Zipchat (chatlive) custom tool — the prompt/instructions, variables,
  channels, and (for gallery templates) the merchant note — that lets the AI
  support agent call an external HTTP API via curl. Trigger whenever someone wants
  to "create/build/draft a custom tool", "connect Zipchat to <API>", "make the bot
  look up orders / create discounts / fetch tracking / escalate tickets", or write
  the instructions/prompt for a custom tool. Produces all the inputs needed to
  install the tool; the author does NOT need access to the chatlive codebase.
---

# Building a Zipchat custom tool

A **custom tool** lets a brand's Zipchat AI agent perform a real action by running a
single `curl` command against an external HTTP API (order lookup, discount creation,
shipment tracking, ticket escalation, etc.). Your job with this skill is to interview
the user and produce **every input needed to install the tool**, with a well-formed
prompt as the centerpiece.

You do not need the chatlive codebase. Everything you need to know about how tools
run is below.

## How a custom tool actually executes (mental model)

1. The brand installs a tool with: a **name**, a **description**, a **prompt**
   (called `instructions`), a set of **variables** (secrets/config), and the
   **channels** it's active on.
2. At conversation time, Zipchat injects the tool's prompt into the agent's system
   prompt, wrapped as:
   ```
   #### <tool name>
   <custom_tool_instructions>
   …your prompt…
   </custom_tool_instructions>
   ```
3. When a customer's request matches the tool, the agent writes **one `curl` command**
   and runs it in a sandboxed shell. The command's output (usually JSON) comes back to
   the agent, which reads it and replies to the customer in the brand's voice.
4. The tool's **variables** are injected into the sandbox as **environment variables**.
   So a variable named `WOO_CONSUMER_KEY` is referenced in the curl as
   `$WOO_CONSUMER_KEY`. Values are encrypted at rest and never shown to the customer.

There is no server-side code per tool — the *only* thing that produces the effect is
the curl hitting the API. If a desired action has no curl, the tool can't do it.

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

### Variable naming rules (enforced)
- Uppercase letters, digits, underscores only; cannot start with a digit. Regex:
  `^[A-Z_][A-Z0-9_]*$`.
- **Must be globally unique across all of a brand's active tools** — so always prefix:
  `WOO_…`, `ZENDESK_…`, `LOYALTY_…`. Never a bare `API_KEY`.

## What platform rules ALREADY cover — do NOT repeat these in your prompt

Zipchat injects shared "Custom Tool Mode" rules into every tool-enabled agent. Your
prompt must **not** restate them (it wastes tokens and drifts out of sync):

- **Curl hygiene** — one command per call, on one line, balanced quotes, no stray
  line breaks in the URL.
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
4. **Channels** — where the tool is active. Valid values: `chat`, `whatsapp`, `email`,
   `instagram`, `messenger`, `zendesk`, `smartlead`, `pdp_questions`, or the sentinel
   `all channels`. Default to `all channels` unless the API/context is channel-specific.
5. **Prompt (`instructions`)** — the core deliverable, ≤10,000 chars. Structure below.
6. **(Optional) Note** — merchant-facing setup instructions for a gallery template
   (e.g. "How to get your REST API key", "what to enter for the store URL").

## Prompt structure

Write the prompt as Markdown with these XML-ish sections, in order. See
`reference/prompt-structure.md` for the full section-by-section guide, and
`examples/` for two complete, real prompts (a read tool and a write tool).

```
<task>            What the tool does + when to use it. If it writes/creates data, say so loudly.
<inputs_resolution>  Every {PLACEHOLDER} and how the agent obtains it (ask customer, read system prompt, decide).
<execution_protocol> The exact curl(s), one per shell call, using $VARS and {PLACEHOLDERS}. State the expected success status.
<tool_persistence_rules>  Anti-hallucination: the agent must observe the real success response before claiming the action happened.
<output_contract>    How to present the result to the customer; what to never reveal (secrets, raw JSON, variable names, the API's name).
```

Add `<composition_rules>` or a tool-specific `<…_rules>` block only when the tool has
real business logic (e.g. a discount cap, a multi-step order). Keep it tight.

## Workflow

1. **Interview.** Ask the user for: the API and the exact endpoint(s) + auth method;
   what the tool should do; what inputs come from the customer vs. stored config; any
   business rules/caps; which channels; and whether this is a one-off install or a
   gallery template (template ⇒ also write a note).
2. **Test the call first if possible.** If you can reach the API, run the real curl
   once to confirm the endpoint, auth, and response shape before writing the prompt.
   A prompt written against a guessed response shape is usually wrong.
3. **Draft** the prompt + all inputs using the structure above.
4. **Validate** against the checklist below.
5. **Deliver** every input clearly labeled so the user can paste them into the
   install form. Variables and the note go separately from the prompt.

## Validation checklist

- [ ] Every secret/brand value is a `$VAR`, never hardcoded; names are prefixed and
      match `^[A-Z_][A-Z0-9_]*$`.
- [ ] Every `{PLACEHOLDER}` has a resolution rule in `<inputs_resolution>`.
- [ ] Exactly one curl per shell call; no `;`, `&&`, `|`, `$(…)`, or inline parsers
      like `jq`/`python -c` (the agent parses JSON by reading it).
- [ ] HTTPS only; the URL is on one line.
- [ ] The prompt does NOT repeat the platform's generic retry/failure/secret rules.
- [ ] `<tool_persistence_rules>` forbids claiming success before the real success
      response is observed (critical for write tools — discounts, escalations, etc.).
- [ ] `<output_contract>` lists what to never reveal: env var values, raw JSON,
      variable names, and the underlying API/vendor name (speak in the brand's voice).
- [ ] Write tools: `<task>` states it creates/modifies data; any tool-specific retry
      nuance (e.g. regenerate a value on conflict) is inline.
- [ ] Channels chosen deliberately; gallery templates include a setup note.
