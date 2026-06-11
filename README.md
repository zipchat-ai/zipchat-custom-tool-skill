# zipchat-custom-tool

A [Claude Code](https://claude.com/claude-code) skill for building **Zipchat custom
tools** — the prompt, variables, and merchant note that let a brand's Zipchat AI support
agent perform a real action (look up an order, create a discount, fetch tracking,
escalate a ticket, or a richer multi-step flow) by running shell commands — `curl` plus
`jq`, `ripgrep`, `fd`, `unzip` — against one or more external HTTP APIs.

You **don't need access to the Zipchat codebase** to use this. The skill embeds
everything about how a custom tool executes, so it can interview you and hand back every
input you paste into the install form.

## What it produces

For any tool you describe, the skill drafts:

1. **Name** — the agent-facing section header.
2. **Description** — one-line gallery/UX copy.
3. **Variables** — the `NAME → what to paste` secrets/config (injected as env vars).
4. **Prompt (`instructions`)** — the core deliverable: a well-formed, structured prompt
   (`<task>`, `<inputs_resolution>`, `<execution_protocol>`, `<tool_persistence_rules>`,
   `<output_contract>`).
5. **(Optional) Note** — merchant-facing setup instructions for a gallery template.

It knows the platform rules already covered by Zipchat (retry/failure/secret hygiene), so
it won't bloat your prompt by repeating them, and it validates the result against a
checklist before handing it over.

## Debugging a tool that doesn't work

If an installed tool misbehaves, the skill will ask you for the **debug transcript**:
in the Zipchat dashboard, find the AI reply (in Conversations or Test chat), click
**Debug** → **RAW debug info** → **Copy debug info**, and paste the result into the
chat. The transcript contains the exact prompts, shell commands, and API responses for
that reply, which the skill reads to pinpoint whether the problem is activation, the
trigger wording, the composed command, or the API response — and hands back a fix.

## Add to Claude Code (plugin — recommended)

Inside a [Claude Code](https://claude.com/claude-code) session, run:

```
/plugin marketplace add zipchat-ai/zipchat-custom-tool-skill
/plugin install zipchat-custom-tool@zipchat-tools
```

Restart Claude Code when prompted. Then invoke the skill by asking Claude to "build a
Zipchat custom tool" / "connect Zipchat to <API>", or run `/zipchat-custom-tool`.

To update later: `/plugin marketplace update zipchat-tools`.

## Add to Claude Code (manual, no plugin)

Clone the repo and copy the skill folder into your personal skills directory:

```
git clone https://github.com/zipchat-ai/zipchat-custom-tool-skill.git
cp -R zipchat-custom-tool-skill/skills/zipchat-custom-tool ~/.claude/skills/
```

It's then available in every project. To scope it to a single project instead, copy it
into that project's `.claude/skills/` directory.

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
