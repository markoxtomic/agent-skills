# multilingual-metadata-agent

An agent skill that produces localized meta titles and meta descriptions for a multilingual Shopify catalog, and writes them back through the Admin GraphQL API.

## Production result

| | |
|---|---|
| Entities | 463 products + 29 collections = 492 |
| Languages | 4 |
| Localized entities | 1,968 |
| Fields produced | 3,936 (meta title + meta description) |
| Deliberately skipped | 4 products whose metadata was already appropriate |
| Manual work avoided | 220+ hours (~7 min per localized entity) |
| Approval gate | Removed after 3–4 calibration runs per language |

Full case study: [Multilingual Metadata Automation](https://www.markotomic.org/systems/multilingual-metadata-automation/)

## Why it works

**Families, not products.** Entities are routed to their product family's context: keywords, terminology, differentiators and tone. One abstraction applied 1,968 times instead of 1,968 prompts.

**Parallel batches.** Subagents run in parallel, each holding one family or a fixed slice of one, in one language. Never one agent working through a list.

**Checkpointing.** Every subagent writes progress to file as it goes. When it hits a rate or credit limit, the run resumes where it stopped.

**An approval gate that is earned away.** Output goes to a human in Excel (accept / reject / blank) until the evidence says it no longer needs to. The gate is removed per language, when rejections stop being about judgment and start being about taste.

## How a run works

```
Configure → Plan batches → Generate (parallel) → Review in Excel → Implement via GraphQL → Report
                                    ↑                    │
                                    └── rejections ──────┘
```

1. **Configure** scope, languages, family map, character limits and approval mode
2. **Plan batches** grouped by family, one batch per subagent
3. **Generate** in parallel, checkpointing incrementally
4. **Review** in one workbook per language. Only accepted rows proceed; blank is never treated as approval
5. **Implement** accepted rows via dry run, batched writes and read-back verification
6. **Report** counts, rejection rate, and whether rejections were about judgment or taste

## The calibration ladder

| Stage | Mode | Move on when |
|---|---|---|
| Runs 1–2 | Full review | Rejections stop being factual or intent errors |
| Runs 3–4 | Spot review | Sampled rejections are about taste, not judgment |
| After | Autonomous | Holds while post-hoc audits stay clean |

Each promotion and demotion is logged with its evidence.

## Before you run it

You need to supply:

- **A family map:** your product families with keyword sets, terminology and tone notes per language. This is the part that carries your domain knowledge; the skill will not start without it.
- **Shopify Admin API access** with write scope for products and collections
- **Your character limits and brand tone.** Defaults are ≤ 70 characters for titles, ≤ 160 for descriptions, calm and informational tone.

Verify the GraphQL mutation names and input shapes against the Admin API version your store runs before the first live write.

## Limitations

- Built and tested on Shopify. Other platforms need a different implementation phase.
- Keyword research is an input, not something the skill does. It routes and writes against the keywords you give it.
- The calibration ladder assumes a human reviewer who knows the market. Skipping to autonomous mode without that loses the main safeguard.

## Files

- [`SKILL.md`](./SKILL.md): the instructions the agent loads

## License

MIT
