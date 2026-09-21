# agent-skills

Production-tested agent skills for e-commerce operations.

Every skill in this repo was run against a real multilingual Shopify catalog before it was published. Each one documents not just what the agent does, but where humans stay in the loop and what evidence it takes to remove them.

## Skills

| Skill | What it does | Production result |
|---|---|---|
| [multilingual-metadata-agent](./multilingual-metadata-agent/) | Localized meta titles and descriptions across languages, with product-family routing, parallel subagents, an Excel approval gate and Shopify GraphQL execution | 3,936 fields · 4 languages · 220+ hours of manual work avoided |

More skills will be added as they graduate from production use.

## Principles

**Tested before published.** No skill lands here on the strength of a demo. Each one has run on real data, and its README says what it produced.

**Autonomy is earned.** Skills that write to live systems start behind a human approval gate. The gate comes off per scope, on evidence, and the evidence is written down.

**Resumable by default.** Agents hit rate limits and die mid-run. Every skill checkpoints to files so a run resumes where it stopped, not from the top.

**No invented facts.** Skills flag missing input rather than filling gaps with plausible-sounding output.

## Using a skill

Each skill is a folder containing a `SKILL.md` (instructions the agent loads) and a `README.md` (for you).

**Claude Code:** copy the skill folder into `~/.claude/skills/` for all projects, or `.claude/skills/` inside a single project.

```bash
git clone https://github.com/markoxtomic/agent-skills
cp -r agent-skills/multilingual-metadata-agent ~/.claude/skills/
```

**Claude.ai:** zip the skill folder and upload it under Settings → Capabilities → Skills.

**Other agents:** the `SKILL.md` is plain Markdown. Load it as a system prompt or instruction file.

Read the skill's README before running it. Most skills need configuration (scope, language list, family map) that only you can supply.

## Structure

```
agent-skills/
├── README.md
├── LICENSE
└── <skill-name>/
    ├── SKILL.md       # agent instructions
    └── README.md      # human documentation
```

## Roadmap

- Eval suites per skill, so you can test a skill against your own data before trusting it
- Synthetic test data via [ecomgen](https://github.com/markoxtomic/ecomgen)

## About

Built by [Marko Tomic](https://www.markotomic.org). Case studies for the systems behind these skills are at [markotomic.org/systems](https://www.markotomic.org/systems/).

## License

MIT
