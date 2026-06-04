# zipchat-custom-tool

A [Claude Code](https://claude.com/claude-code) skill for building **Zipchat custom
tools** — the prompt, variables, channels, and merchant note that let a brand's Zipchat
AI support agent perform a real action (look up an order, create a discount, fetch
tracking, escalate a ticket) by calling an external HTTP API via `curl`.

You **don't need access to the Zipchat codebase** to use this. The skill embeds
everything about how a custom tool executes, so it can interview you and hand back every
input you paste into the install form.

## What it produces

For any tool you describe, the skill drafts:

1. **Name** — the agent-facing section header.
2. **Description** — one-line gallery/UX copy.
3. **Variables** — the `NAME → what to paste` secrets/config (injected as env vars).
4. **Channels** — where the tool is active (`chat`, `whatsapp`, `email`, … or `all channels`).
5. **Prompt (`instructions`)** — the core deliverable: a well-formed, structured prompt
   (`<task>`, `<inputs_resolution>`, `<execution_protocol>`, `<tool_persistence_rules>`,
   `<output_contract>`).
6. **(Optional) Note** — merchant-facing setup instructions for a gallery template.

It knows the platform rules already covered by Zipchat (retry/failure/secret hygiene), so
it won't bloat your prompt by repeating them, and it validates the result against a
checklist before handing it over.

## Install (as a Claude Code plugin)

```
/plugin marketplace add Leteyski/zipchat-custom-tool-skill
/plugin install zipchat-custom-tool@zipchat-tools
```

Then invoke it by asking Claude to "build a Zipchat custom tool" / "connect Zipchat to
<API>", or run `/zipchat-custom-tool`.

## Install (manual, no plugin)

Copy the skill folder into your personal skills directory:

```
cp -R skills/zipchat-custom-tool ~/.claude/skills/
```

Available in every project. Or drop it into a single project's `.claude/skills/` instead.

## What's inside

```
skills/zipchat-custom-tool/
├── SKILL.md                       # entry point: execution model, inputs, workflow, checklist
├── reference/prompt-structure.md  # section-by-section prompt guide
└── examples/
    ├── order-lookup.md            # worked read-only tool (WooCommerce order + tracking)
    └── create-discount.md         # worked write tool (coupon creation, safety cap)
```

## License

MIT — see [LICENSE](LICENSE).
