# retirement-income (Claude Agent Skill)

Plan retirement decumulation: tax-smart withdrawal order, dynamic/variable spending strategies (Guyton-Klinger guardrails, VPW, spending smile, bucket+buffer vs static 4% SWR), Social Security claiming age, ACA healthcare bridge before Medicare, projected RMDs with a tax-torpedo / Roth-conversion-runway readout (`analyze_rmd`), Medicare IRMAA tier and surcharge-cliff lookup (`analyze_irmaa`), Medicare enrollment-timing windows and permanent late-enrollment penalties (`analyze_medicare_enrollment`), guaranteed income, a bond/TIPS ladder income floor, long-term-care cost exposure, and estate-tax exposure. Thin orchestration over the planfi MCP.

It's a **thin orchestration layer** over the public **planfi MCP** (`https://ai.planfi.app/mcp`,
public, no auth) — all the math and financial logic live server-side. The skill itself bundles no
engine; it just gathers inputs and calls the tools.

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp` (no auth). The skill also reminds you to do this if the tools are
missing when you invoke it.

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-retirement-income
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/retirement-income ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/retirement-income .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-retirement-income
/plugin install retirement-income@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r retirement-income.zip retirement-income`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports — each tool returns a structured
  `assumed_defaults[]` array of `{ field, assumed_value, note }` — so you can correct any silent
  default.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-retirement-income>.

## Use it in any MCP client (not just Claude Code)

This skill is Claude Code packaging — but the engine is a standard
[Model Context Protocol](https://modelcontextprotocol.io) server at
`https://ai.planfi.app/mcp` (Streamable HTTP, no auth). Connect it from any
MCP-capable agent and you get the same PlanFi tools directly. Every tool is
**self-orchestrating** (it reports its own assumed defaults and suggests the
next step), so it works well even without the skill wrapper.

Most clients take an `mcpServers` config block:

```json
{
  "mcpServers": {
    "planfi": { "type": "http", "url": "https://ai.planfi.app/mcp" }
  }
}
```

| Client | How to add it |
|--------|---------------|
| **Claude Code** | `claude mcp add --transport http planfi https://ai.planfi.app/mcp` |
| **Cursor** | add the block above to `~/.cursor/mcp.json` (field: `url`) |
| **Windsurf** | `~/.codeium/windsurf/mcp_config.json` (field: `serverUrl`) |
| **Cline / VS Code** | paste the block into the Cline MCP settings |
| **Claude Desktop & stdio-only clients** | bridge with `npx -y mcp-remote https://ai.planfi.app/mcp` |
| **ChatGPT (custom connectors / Deep Research)** | add a connector pointing at the MCP URL |
| **Custom / your own agent** | plain MCP Streamable HTTP — POST JSON-RPC `tools/list` / `tools/call` to the URL with `Accept: application/json, text/event-stream` |

Field names vary slightly by client and version — check your client's MCP docs;
the URL is always `https://ai.planfi.app/mcp`.

## License

MIT — see [LICENSE](./LICENSE).
