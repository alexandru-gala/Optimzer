---
name: pago-optimizer
description: Personal-productivity audit for Claude usage. Asks 7 quick questions about your habits, then returns 3 ranked recommendations to slow your usage-window burn. Honest framing — this is a heuristic checklist, not analysis of your actual session data. For real telemetry, the org admin maintains a separate dashboard from Office/Cowork OpenTelemetry events.
when_to_use: When a Pago colleague says their usage window is burning too fast, before they upgrade their plan, or as a quarterly habit check-in. Run this before assuming the problem is Anthropic's quotas vs. your workflow.
user-invocable: true
---

# Pago Optimizer

You are running a 7-question structured intake for a Pago colleague who feels their Claude usage window is burning too fast. You will then return exactly 3 concrete recommendations ranked by conditional impact.

## Critical honesty rules — non-negotiable

1. Do NOT claim you are "analyzing the user's last 7 or 14 days" — you have no telemetry access from inside this surface (claude.ai chat or Office/Cowork agent). You are running a self-report intake. State this plainly at the start.
2. Every percentage savings figure you quote MUST include the condition under which it holds. "Sonnet is ~40% cheaper than Opus per input token, applied to the calls where you switch" — not "save 40%."
3. Cite Anthropic docs for each claim, and tag the source tier: `[vendor]` for Anthropic itself, `[community-single]` for unreplicated community sources, `[community-replicated]` for multiply-corroborated, `[independent]` for peer-reviewed or independent measurement.
4. Never suggest a recommendation that depends on a feature unavailable in the user's surface (e.g., do not suggest hooks to a claude.ai chat user — hooks are Claude Code only).
5. End every output with the disclaimer in the OUTPUT TEMPLATE section.

## Step 1 — Detect surface

Ask the user once: "Are you running this in claude.ai chat, an Office/Cowork agent, or Claude Code CLI?" Their answer changes which recommendations are eligible (see ELIGIBILITY MATRIX).

## Step 2 — Run the intake

Ask exactly these 7 questions, one at a time. Wait for the answer before moving on. For forced-choice questions, present the options inline.

1. **Default model for new tasks?** [Haiku 4.5 / Sonnet 4.6 / Opus 4.7 / I don't pick explicitly]
2. **Typical session length before you /clear or open a new chat?** [<10 messages / 10–30 / 30–80 / 80+ / I never /clear]
3. **Extended thinking default?** [off / low / high / leave on always / don't know]
4. **Largest single attachment in the past week?** [<5KB / 5–50KB / 50–500KB / 500KB+ / I attach the full repo]
5. **MCP servers active right now?** (free text, comma-separated, or "none / don't know")
6. **Most repeated multi-step task this week?** (one line, free text)
7. **Biggest frustration moment this week?** (one line, free text — what made you feel "this is wasting my window")

If the user skips or gives unusable answers to >3 questions, switch to the SPARSE INTAKE FALLBACK at the bottom.

## Step 3 — Match recommendations

Run the decision tree below in order. Each rule fires at most once. Pick the first 3 that match. If fewer than 3 match, fill remaining slots from the GENERIC TOP 3.

### Decision tree (priority order)

| Rule | Trigger | Recommendation | Conditional impact | Source tier |
|---|---|---|---|---|
| R1 | Q1=Opus AND (Q2 ≥ 30 OR Q2=never) | Switch default to Sonnet for coding; reserve Opus for architecture/planning sessions only | Sonnet input ≈ 40% cheaper than Opus per token; impact = (40% × fraction of your calls that move). Per the Anthropic pricing page. | [vendor] |
| R2 | Q3 ∈ {leave on always, high} AND Q6 contains routine work | Disable extended thinking for routine tasks; turn it on per-task only when reasoning depth matters | Thinking tokens bill as output. For tasks where thinking adds no value, disabling removes the thinking-token output overhead on those calls. Per Anthropic extended-thinking docs. | [vendor] |
| R3 | Q2 ∈ {30–80, 80+, never /clear} | Adopt /clear discipline between unrelated tasks; treat new task = new context | Each retained message reprocesses prior context. Past msg ~50, per-message input cost grows roughly linearly. Behavioral change, no published % figure. Per Anthropic best-practices. | [vendor] |
| R4 | Q4 ∈ {500KB+, full repo} | Use subagents to do codebase exploration; only the summary returns to your main thread | Subagent context is isolated from main; effective context retained = main_size − summary_size. Magnitude depends on exploration depth. Per Anthropic best-practices. | [vendor] |
| R5 | Q5 lists 3+ servers AND user can't justify ≥2 | Disable unused MCP servers in settings | Each loaded MCP adds tool-definition tokens to every message. Magnitude depends on definitions size; not all servers are equal. Per Anthropic costs doc. | [vendor] |
| R6 | Q1=I don't pick explicitly | Pin a model default per task type (Haiku for triage, Sonnet for code, Opus for architecture) | Without an explicit pick the platform may route to a more expensive model. Pricing differential is the savings ceiling. Per Anthropic pricing. | [vendor] |
| R7 | Q6 = clearly templatable repeated task | Convert the repeated task into a saved skill or prompt template | Skills load on demand and are cached as system content; structural improvement, not a % savings claim. Per Anthropic skills docs. | [vendor] |
| R8 | Q7 contains "context full" / "compacted" / "ran out" | Move heavy reference material from CLAUDE.md to on-demand skills | CLAUDE.md content past ~200 lines is loaded every session start; skills load only when triggered. Per Anthropic best-practices. | [vendor] |

### Eligibility matrix (filter recommendations by surface)

| Recommendation | claude.ai chat | Office/Cowork | Claude Code |
|---|---|---|---|
| R1 model routing | yes (in-chat /model) | yes | yes |
| R2 thinking off | yes | yes | yes |
| R3 /clear discipline | yes | yes (new agent run) | yes |
| R4 subagents | partial (limited) | yes | yes |
| R5 disable MCPs | yes (Settings → Connectors) | yes (Settings → Connectors) | yes (.claude/settings.json) |
| R6 pin model | yes | yes | yes (`/model` + per-skill `model:` field) |
| R7 skill / template | yes (org skill) | yes (plugin) | yes |
| R8 CLAUDE.md → skill | NA (no CLAUDE.md) | NA | yes |

If a triggered rule isn't eligible for the user's surface, skip it and try the next match.

### Generic top 3 (fallback if intake is too sparse, with explicit disclaimer)

1. Adopt /clear discipline between unrelated tasks. (R3)
2. Pin Sonnet as your coding default; use Opus only for architecture. (R1/R6)
3. Disable extended thinking for routine work. (R2)

State explicitly: "Your intake was too sparse for personalized recos — these are the generic top 3 most Pago users benefit from."

## Step 4 — Output template

Render exactly this format. Three recos, no more, no less. After all three, the disclaimer block is required.

```
PAGO OPTIMIZER — 3 recommendations for you
Surface detected: <claude.ai chat | Office/Cowork | Claude Code>
Based on: self-report intake (7 questions). Not telemetry.

—— RECOMMENDATION 1 of 3 ——
Action: <one imperative line>
Why this is #1 for you: <which intake answer triggered it, in plain language>
Conditional impact: <savings figure WITH the condition that makes it real>
Source: <doc URL> [tier]
Do this today (30 seconds): <concrete step>

—— RECOMMENDATION 2 of 3 ——
[same shape]

—— RECOMMENDATION 3 of 3 ——
[same shape]

—— Honest disclaimer ——
These are heuristics from a 7-question intake, not analysis of your actual usage data.
Real per-user analysis would require telemetry that this surface does not expose.
For org-wide trends, ask alex@pago.ro about the Pago telemetry dashboard
(opt-out available; events are pseudonymized at ingest).
```

## Step 5 — Do NOT

- Do not save the user's answers anywhere. They live for one turn only.
- Do not POST or transmit anything. The skill has no telemetry side channel — telemetry is org-admin infrastructure, separate from this skill.
- Do not claim "X% savings" without the condition.
- Do not suggest hooks, PreToolUse filters, or Claude Code-only features to claude.ai or Office/Cowork users.
- Do not run the intake more than once per invocation.
- Do not overshoot 3 recommendations. Three. Always three.

## Sparse intake fallback

If user answered <4 questions usefully, output: "I don't have enough to personalize. Here are the 3 things most Pago users benefit from — adopt the one that matches your gut feel about where your time goes." Then emit the GENERIC TOP 3 with the same template.

## Maintenance notes (for future editors of this skill)

- Quarterly: re-verify all R1–R8 source URLs and pricing figures. If Anthropic pricing inverts (e.g., Sonnet > Opus), R1's claim falsifies — update or remove.
- The "v2.1.x cache bug" recommendation (community-single source) is intentionally NOT in R1–R8. It stays as a footnote in `sources.md` until corroborated.
- "2-4x velocity uplift" is intentionally NOT claimed anywhere. Selection-effect risk; not falsifiable from a 7-question intake.
- Keep this file under 500 lines per the skills-format guidance. Move long reference material into `recommendations.md` and `sources.md`.
