# Pago Optimizer — Shared Recommendation Library

**Single source of truth.** Both the chat SKILL.md and the Cowork plugin import recommendations from this file. Edit here, propagate to both surfaces in CI.

## Schema

Each recommendation has:

- `id`: stable identifier (R1, R2, …) — never reuse, never renumber
- `action`: imperative one-liner the user does
- `triggers`: intake-answer predicates that fire this rec
- `base_impact`: high / medium / low — the impact ceiling assuming the user's burn matches this pattern
- `confidence`: vendor-pricing / telemetry-backed / self-report — what backs the impact claim
- `eligible_surfaces`: which surfaces can actually act on it
- `why_template`: format string with `{Q1}`, `{Q2}`, etc. slots from the intake
- `today_step`: 30-second concrete action
- `source`: doc URL + tier marker
- `falsified_when`: condition under which this recommendation is no longer valid

## Scored ranking

Each rec emits a score: `score = base_impact_weight × confidence_weight × eligibility`.

| `base_impact` | weight |
|---|---|
| high | 3 |
| medium | 2 |
| low | 1 |

| `confidence` | weight |
|---|---|
| vendor-pricing | 3 |
| telemetry-backed | 3 |
| self-report | 2 |

`eligibility` is 1 if the user's surface supports the action, 0 otherwise. Score 0 → drop.

Top 3 by score. Ties broken by `base_impact_weight` then `confidence_weight` then `id` ascending.

## The recommendations

### R1 — Switch coding default to Sonnet
- triggers: `Q1 == Opus AND (Q2 in {30-80, 80+, never} OR Q7 in {most, always})`
- base_impact: high
- confidence: vendor-pricing
- eligible_surfaces: chat, cowork
- why_template: "You said you default to Opus and your sessions are long / tool-heavy. Sonnet is meaningfully cheaper per input token; for the same task profile this maps directly to lower spend."
- today_step: "/model claude-sonnet-4-6 in your next chat. Reserve Opus for architecture sessions only."
- source: https://platform.claude.com/docs/en/about-claude/pricing — [vendor]
- falsified_when: Anthropic pricing inverts (Sonnet ≥ Opus)

### R2 — Disable extended thinking for routine work
- triggers: `Q3 in {high, always-on} AND Q6 in {code, docs, mixed}`
- base_impact: medium
- confidence: vendor-pricing
- eligible_surfaces: chat, cowork
- why_template: "You said extended thinking is on by default and your work is mostly routine code / docs. Thinking tokens bill as output; on routine calls that overhead has no payoff."
- today_step: "/effort low (or off) for routine tasks. Re-enable per-task only when reasoning depth matters."
- source: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/extended-thinking-tips — [vendor]
- falsified_when: Anthropic stops billing thinking tokens as output

### R3 — Adopt /clear discipline between unrelated tasks
- triggers: `Q2 in {30-80, 80+, never}`
- base_impact: high
- confidence: self-report (no published % figure)
- eligible_surfaces: chat, cowork
- why_template: "You said sessions go {Q2} messages. Each retained message reprocesses prior context; past msg ~50, per-message input cost grows roughly linearly."
- today_step: "/clear (chat) or 'New agent' (Cowork) before each unrelated task. New task = new context."
- source: https://code.claude.com/docs/en/best-practices — [vendor]
- falsified_when: never (this is structural to how transformer context works)

### R4 — Use subagents for codebase exploration / large-file work
- triggers: `Q4 in {500KB+, full-repo} OR Q7 in {most, always}`
- base_impact: medium
- confidence: self-report (depends on exploration depth)
- eligible_surfaces: cowork (full), chat (limited — server-side tools only)
- why_template: "You said you {Q4 or Q7-driven phrase}. A subagent's context is isolated from your main thread; you receive the summary only."
- today_step: "Tell the agent: 'Use a subagent to explore this codebase and report back.' Or in Cowork, kick off a side task."
- source: https://code.claude.com/docs/en/best-practices — [vendor]
- falsified_when: subagent feature deprecated

### R5 — Disable unused MCP servers
- triggers: `Q5 in {3-5, 6+}`
- base_impact: medium
- confidence: self-report (depends on definitions size)
- eligible_surfaces: chat, cowork
- why_template: "You said you have {Q5} MCP servers active. Each loaded MCP adds tool-definition tokens to every message — even if never invoked."
- today_step: "Settings → Connectors. Disable any server you can't justify by name. Audit again next week."
- source: https://code.claude.com/docs/en/costs — [vendor]
- falsified_when: never (architectural)

### R6 — Pin a model default per task type
- triggers: `Q1 == no-explicit-pick OR Q8 == always-default`
- base_impact: medium
- confidence: vendor-pricing
- eligible_surfaces: chat, cowork
- why_template: "You said you don't explicitly pick a model. Without a pick, the platform may route to a more expensive model than your task needs."
- today_step: "Pick: Haiku for triage / quick edits, Sonnet for code, Opus only for architecture. Set Sonnet as your chat default."
- source: https://platform.claude.com/docs/en/about-claude/pricing — [vendor]
- falsified_when: Anthropic introduces transparent auto-routing

### R7 — Convert repeated work into a saved skill or template
- triggers: `Q9 contains repeated-task-language (free-text heuristic)`
- base_impact: medium
- confidence: self-report (structural improvement, no % figure)
- eligible_surfaces: chat (org skill), cowork (plugin)
- why_template: "Your frustration mentioned: '{Q9}'. If that's a repeated multi-step task, lifting it into a saved skill cuts the per-invocation prompt cost and keeps the recipe stable."
- today_step: "Sketch the recipe in 5 lines. Ask alex about the org skill registry, or save as a chat template."
- source: https://code.claude.com/docs/en/skills — [vendor]
- falsified_when: skill format deprecated

### R8 — Move heavy reference material from CLAUDE.md to on-demand skills
- triggers: `Q9 contains "context full" / "compacted" / "ran out"`
- base_impact: medium
- confidence: self-report (structural)
- eligible_surfaces: claude-code only (chat / Cowork have no CLAUDE.md)
- why_template: "Your frustration matched 'context bloated'. CLAUDE.md content past ~200 lines loads every session start; on-demand skills load only when triggered."
- today_step: "Audit CLAUDE.md. Anything that's domain-specific or rare → move into a named skill, leave just the always-applicable rules in CLAUDE.md."
- source: https://code.claude.com/docs/en/best-practices — [vendor]
- falsified_when: never (structural)

## Footnotes (NOT surfaced as recommendations)

- **Claude Code v2.1.x prefix-cache bugs (10–20× inflation)**: Single-source citation (theregister.com 2026-03-31). DEMOTED. Mention only in `sources.md`, not in the skill output. Re-evaluate if independently corroborated.
- **"2–4× velocity uplift" claims**: Marketing-adjacent / selection-effect risk. NOT included. Not falsifiable from a 9-question intake.

## Generic top 3 (sparse-intake fallback)

If the user gave usable answers to <5 of the 9 questions, output these instead of personalized:

1. R3 — adopt /clear discipline
2. R6 — pin a model default
3. R2 — disable extended thinking for routine work

State explicitly in the output: "Your intake was too sparse for personalized recommendations — these are the three habits most Pago users benefit from."
