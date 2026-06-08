# startup-equity-planner (Claude Agent Skill)

Model the two highest-leverage pre-IPO/startup equity moves — the irreversible 30-day 83(b)
early-exercise election (ordinary→LTCG conversion + holding-clock start) and the §1202 QSBS gain
exclusion (up to $10M legacy / $15M OBBBA, or 10× basis) — for startup founders, early employees,
pre-IPO equity holders, and C-corp business sellers, by orchestrating the public planfi MCP.

**Related skill:** for broader RSU/ISO/NSO/ESPP grant valuation and single-stock concentration, see
[equity-comp-planner](https://github.com/holdequity/planfi-equity-comp-planner).

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
npx skills add holdequity/planfi-startup-equity-planner
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/startup-equity-planner ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/startup-equity-planner .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-startup-equity-planner
/plugin install startup-equity-planner@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r startup-equity-planner.zip startup-equity-planner`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports — `analyze_startup_equity` returns a
  structured `assumed_defaults[]` array (`{ field, assumed_value, note }`) so you can correct any
  silent default (plus `disclosures.key_assumptions` methodology prose).
- The 83(b) election is **irreversible** and must be filed within **30 days of exercise** — the skill
  always surfaces this caveat.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-startup-equity-planner>.

## License

MIT — see [LICENSE](./LICENSE).
