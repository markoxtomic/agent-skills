---
name: multilingual-metadata-agent
description: Produce and implement localized SEO metadata (meta titles and meta descriptions) for a multilingual Shopify catalog at scale, using product-family routing, batched parallel subagents with file checkpointing, a human Excel approval gate, and Shopify Admin GraphQL execution. Use this skill whenever the user mentions metadata, meta titles, meta descriptions, SEO fields, product families, catalog-wide SEO, multilingual or localized product metadata, bulk metadata generation, metadata review or approval, or asks to write, audit, translate or push metadata for an online store — even if they never say "SEO". Also use it when the user pastes a product export and asks what the titles and descriptions should be, or asks how to roll out metadata across several languages or markets.
---

# Multilingual Metadata Agent

Produces localized meta titles and meta descriptions for a Shopify catalog across multiple languages, and writes them back via the Admin GraphQL API.

The system exists because per-entity prompting does not scale and generic AI output is worse than an empty field. Two things make it work: **routing by product family instead of by product**, and **an approval gate that is removed only after it has been earned**.

Read this whole file before starting. The phases are sequential and the gates are not optional.

---

## Core principles

**1. Families, not products.** One abstraction applied to thousands of entities beats thousands of bespoke prompts. Every entity is routed to its family's context — terminology, keyword intent, tone, differentiators. If an entity does not fit a family, stop and ask; do not improvise a family.

**2. Localization is not translation.** Each language gets its own keyword intent and terminology, built from how people search in that market. Never translate a keyword literally across languages. This is why calibration happens per language rather than once.

**3. Autonomy is earned per language.** The approval gate is removed only after the evidence supports it, language by language, and the evidence is written down. Never assume approval can be skipped because it was skipped for another language.

**4. Leaving a field alone is a valid output.** Existing metadata that is already good must be recognized and skipped, with a reason recorded. A run where nothing is skipped is a suspicious run.

**5. Never invent product facts.** Dimensions, materials, certifications, delivery times, stock and prices come from the input only. A flagged gap beats a plausible invention.

---

## Phase 0 — Configure the run

Collect before anything else:

- **Scope:** which entity types (products, collections), which IDs, which languages
- **Family map:** the product families and, per family, their keyword set, terminology, differentiators and tone notes
- **Character limits:** meta title ≤ 70 characters, meta description ≤ 160 characters (confirm against the store's current rules)
- **Brand tone:** the default is calm and informational — no hype, no exclamation marks, no aggressive commercial phrasing
- **Approval mode:** `full-review`, `spot-review`, or `autonomous` (see the calibration ladder)
- **Batch size** and **subagent count**

Refuse to start if the family map or the language list is missing. Guessing either produces output that looks right and is wrong at scale.

Write the resolved configuration to `run/config.json` so the run is reproducible and resumable.

---

## Phase 1 — Plan batches

1. Group entities by product family.
2. Split each family into batches sized so one batch fits comfortably inside a single subagent's context and rate-limit budget. A family larger than the batch size is split into fixed slices; a batch never spans two families.
3. Write the batch plan to `run/batches/<language>/<family>-<slice>.json` with entity IDs, family context and target language.
4. Each batch gets a status file: `pending`, `in_progress`, `complete`, `failed`.

Batches are the unit of work, of checkpointing and of resumption. Never generate outside a batch.

---

## Phase 2 — Generate (parallel subagents)

Dispatch batches to subagents in parallel. Each subagent holds exactly one batch — one family, or one fixed slice of a family — in one language.

**Every subagent must:**

- Write generated rows to its output file **incrementally**, not at the end of the batch
- Update its status file after every N entities
- Record the last completed entity ID so a resumed run continues from there, not from the top
- Write a short run note (what it did, what it skipped, what it flagged) alongside the output

**Checkpointing is mandatory, not defensive.** Subagents hit rate and credit limits mid-run. A batch that dies without a checkpoint costs the whole batch. On restart, read every status file first and resume only the `in_progress` and `failed` batches.

**Per entity, the subagent produces:**

| Field | Rule |
|---|---|
| `meta_title` | ≤ 70 characters. Family keyword aligned. Unique across the catalog. |
| `meta_description` | ≤ 160 characters. One concrete differentiator. Not a restatement of the title. |
| `action` | `write`, `skip` (existing metadata already appropriate), or `flag` (insufficient input) |
| `reason` | Required for `skip` and `flag` |

**Count characters, do not estimate.** Report the count with each field. German and French run 20–30% longer than English and will breach the limit if the budget is not re-checked after localization.

**Duplicate detection runs across the whole catalog**, not within the batch. Maintain a shared index of produced titles and descriptions; flag near-duplicates rather than shipping them.

---

## Phase 3 — Human approval gate (Excel)

Unless the run is configured `autonomous` for this language, generation output goes to a human before any write.

**Export one workbook per language**, one row per entity, with these columns:

| Column | Purpose |
|---|---|
| `entity_id` | Product or collection ID |
| `entity_type` | product / collection |
| `family` | Routed family |
| `current_title` / `current_description` | What is live now |
| `proposed_title` / `proposed_description` | What the agent produced |
| `title_length` / `description_length` | Character counts |
| `agent_action` | write / skip / flag |
| `agent_reason` | Why, for skip and flag |
| `decision` | **Human fills this in** |
| `correction` | **Human fills this in** |

**The human decision vocabulary is exactly three values:**

- **Accept** — implement as proposed
- **Reject** — do not implement; regenerate with the correction note
- **Blank** — treated as not reviewed; never implemented, carried to the next round

Blank is not implicit approval. Treat an unreviewed row as a hard block on writing that entity.

**Read the workbook back** and partition rows into accepted, rejected-with-correction, and unreviewed. Feed rejections and their correction notes back into the generating subagent for that family, then re-export. Only accepted rows proceed.

Log every round to `run/review/<language>-round-<n>.xlsx` and keep them. These files are the evidence for the calibration ladder.

---

## Phase 4 — Implement via Shopify Admin GraphQL

Only accepted rows are written. Nothing else reaches the store.

**Order of operations:**

1. **Dry run first.** Produce the exact mutation payload for every row and validate it without sending. Report the count and any payload that fails validation.
2. **Write in batches**, respecting the API's cost-based rate limiting. Back off on throttle responses rather than retrying immediately.
3. **Checkpoint after every batch** — which entity IDs were written, with the response status. A killed implementation run must resume without rewriting what already succeeded.
4. **Verify by reading back** a sample of written entities and comparing the live values against what was sent. Report any mismatch.

**Mutations:**
- Products: `productUpdate` with the `seo { title, description }` input
- Collections: `collectionUpdate` with the same `seo` input
- Localized values: write through the translation API for each non-primary locale rather than overwriting the primary-locale field

> Verify the exact mutation names, input shapes and API version against the Shopify Admin API docs for the version the store is on before the first live write. API shapes change between versions, and a wrong field name fails silently on some clients.

**Never** write to an entity whose row is blank, rejected, or flagged. **Never** write outside the configured scope, even if an obviously broken entity turns up next to one in scope — report it instead.

---

## Phase 5 — Report

At the end of every run, write `run/report-<language>.md` containing:

- Entities in scope, written, skipped, flagged, blocked by review
- Rejection rate this round, and the trend across rounds
- What the rejections were about: **judgment** (the agent got the intent wrong) or **taste** (the human would have phrased it differently)
- Duplicate flags raised
- Batches that failed and resumed
- Verification sample result

The judgment-versus-taste split is the important one. It is the input to the next decision.

---

## The calibration ladder

The approval gate is removed per language, on evidence, in this order:

| Stage | Mode | Exit condition |
|---|---|---|
| Runs 1–2 | Full review — every row read by a human | Rejection rate falls and rejections stop being about factual or intent errors |
| Runs 3–4 | Spot review — a sample read by a human | Sampled rejections are about taste, not judgment |
| Thereafter | Autonomous — no gate | Holds only while post-hoc audits stay clean |

**The rule for removing the gate:** when the rejections stop being about judgment and start being about taste, the gate has stopped doing work. Not before.

Record the evidence for each promotion in `run/autonomy-log.md`: which language, which run, the rejection rate, and the reasoning. Autonomy that is not written down is not autonomy, it is drift.

**Demote on evidence too.** If a post-hoc audit turns up judgment errors in an autonomous language, that language goes back to spot review and the log says why.

---

## Quality rules

**Meta title**
- ≤ 70 characters including spaces
- Family keyword present and natural in the target language
- Unique across the whole catalog, across all languages
- No ALL CAPS, no keyword stuffing, no more than two separators

**Meta description**
- ≤ 160 characters
- Contains the family keyword once, naturally
- Contains one concrete differentiator — material, dimension, construction, provenance — not a generic claim
- Does not restate the title
- Does not duplicate the product body copy

**Tone**
- Calm and informational. The brand is premium, not promotional.
- No hype, no urgency, no exclamation marks
- No unverifiable superlatives
- Locale-correct register, spelling and number formatting (decimal commas, `ß` vs `ss`, formal vs informal address)

---

## Anti-patterns

- Prompting per product instead of routing by family
- Running one agent serially over the whole catalog
- Writing output only at the end of a batch
- Treating a blank review cell as approval
- Removing the approval gate for all languages because one language earned it
- Writing to the store before a dry run
- Translating a keyword literally into another language
- Padding a field to hit a character minimum
- Inventing an attribute to make a description more specific

---

## File layout

```
run/
├── config.json
├── batches/<language>/<family>-<slice>.json
├── status/<language>/<family>-<slice>.status
├── generated/<language>/<family>-<slice>.jsonl
├── review/<language>-round-<n>.xlsx
├── implemented/<language>-<batch>.json
├── report-<language>.md
└── autonomy-log.md
```

Everything under `run/` is resumable state. A run that cannot be killed and restarted is not finished being built.
