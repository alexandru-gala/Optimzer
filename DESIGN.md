# Pago Optimizer Skill — Design v2

**Status**: Draft after research + Kimi critique + OTel screenshot unlock. Awaiting codex review.
**Owner**: alex@pago.ro
**Date**: 2026-05-01

## Goal

Ship a skill to Pago colleagues using Claude.ai chat and Office/Cowork agents that helps them stop burning their usage windows so fast. Side channel: anonymized org-wide telemetry to Pago so we can see which patterns dominate and tune the advice.

## What changed since v1

1. **OTel telemetry confirmed**: The Team plan exposes an OTLP endpoint configurable by org admins. Office/Cowork agent events stream to a Pago-controlled collector. This eliminates the need for any client-side telemetry sidecar in the skill.
2. **Kimi flagged 5 epistemic problems** in v1 — fraudulent "7d/14d analysis" framing, unconditional savings figures, "anonymous" misnomer, citation-chain fragility, and selection-effect hand-waving on velocity claims. v2 fixes these.
3. **Tier B (bundled MCP telemetry connector for chat users) dropped**. Privacy weakest link, removed entirely.

## Hard constraints (verified)

- Hooks are Claude Code CLI only — neither claude.ai nor Office/Cowork support them.
- Office/Cowork has no raw SKILL.md drop-in — skills ship as plugins. claude.ai chat installs SKILL.md directly via Settings.
- No filesystem access from claude.ai chat skills.
- Console Usage API requires admin key — not callable from inside a skill.
- claude.ai chat history export is human-initiated zip — not automatable.
- Office/Cowork OTel events include `user.email`, `organization.id`, `session.id`, `prompt.id`, tool/MCP calls, file access, approval decisions, tokens, costs.

## Architecture

### Layer 1 — The skill (per-user, runs in chat or Office agent)

`pago-optimizer` skill, distributed via org-shared Skills (Settings > Skills > Share with organization).

**What it does, honestly:**
- Runs a 7-question structured intake (NOT analysis) about the user's habits
- Maps answers to a decision tree drawn from the top-10 known causes of fast quota burn
- Outputs 3 recommendations ranked by *conditional* impact (each estimate states the condition under which it applies)
- For each reco: the action, the link to the skill/plugin/template that enables it, the *condition* under which the savings figure applies, and a 1-line "do this today" step

**What it deliberately does NOT do:**
- Claim to "analyze your last 7d/14d" without telemetry
- Quote "89.5% savings" without "if your cache hit rate is X% and your prefix is stable"
- Self-identify as "AI productivity coach" — it's a checklist with a brain

**Surface compatibility:**
- claude.ai chat: SKILL.md install, intake-only, no telemetry from skill
- Office/Cowork: same SKILL.md packaged as a plugin, intake-only, telemetry flows independently via Layer 2
- Claude Code (bonus): same SKILL.md drop-in to `~/.claude/skills/`, can additionally read `~/.claude/projects/*.jsonl` for real session analysis (then framing becomes honest)

### Layer 2 — Telemetry (org-admin, runs at Pago)

OTLP collector at Pago receives Office/Cowork events. **Anonymization at ingest, not in transit:**

```
[Cowork client] --OTLP/HTTPS--> [pago-otel-collector] --filtered--> [pago-otel-store]
                                       |
                                       +-- HMAC-SHA256(user.email, $SECRET) -> user_pseudonym
                                       +-- DROP user.email
                                       +-- DROP prompt.text, prompt.id (re-ID risk)
                                       +-- KEEP: model, tool_name, mcp_server_name,
                                                 input_tokens, output_tokens,
                                                 cache_read_tokens, cache_creation_tokens,
                                                 duration_ms, timestamp_day (not exact ts)
                                       +-- AGGREGATE to daily buckets, k>=5 before exposure
```

**Collector tech**: open-source `otelcol` with a custom processor for the HMAC + drop rules. Ship behind Cloudflare Access for ingress auth. Output store: Postgres or ClickHouse — small enough that Postgres is fine for 50 users.

**Honest privacy framing**: this is **pseudonymized**, not anonymous. At N=50 per week, joining `user_pseudonym + plan_tier + recommendation_ids` to one side fact (e.g., onboarding date) re-identifies. We commit to: never publish per-pseudonym detail outside k>=5 cohorts; rotate the HMAC secret quarterly; document the re-ID risk in the team comms when announcing the skill.

### Layer 3 — Dashboard (alex-only, Pago internal)

Read from collector store. Surfaces:
- Org-wide top 5 burn patterns (e.g., "62% of sessions use Opus when last tool call was a trivial edit")
- Recommendation adoption signal (did `model: haiku` show up in subagent calls after the skill recommended it? — coarse-grained, no per-user identification)
- Cache hit rate distribution
- Distribution of MCP servers loaded vs. invoked (idle MCPs)

This isn't a productivity surveillance tool. It's an aggregate signal so I can update the skill's decision tree based on what's *actually* dominant at Pago vs. what's loud on Twitter.

## The intake (Layer 1 detail)

7 questions, all forced-choice or numeric. Asked once per skill invocation. No memory across sessions (privacy + simplicity).

1. **Default model for new tasks?** [Haiku / Sonnet / Opus / I-don't-pick-explicitly]
2. **Typical session length before /clear or new chat?** [<10 msgs / 10-30 / 30-80 / 80+ / never /clear]
3. **Extended thinking default?** [off / low / high / leave-on-always / don't-know]
4. **Largest single attachment in the past week?** [<5KB / 5-50KB / 50-500KB / 500KB+ / repeated full codebase]
5. **MCP servers active right now?** (free text, comma-sep) — used to flag idle MCPs
6. **Most repeated task this week?** (free text, 1 line) — used to suggest a custom skill or template
7. **Frustration moment this week?** (free text, 1 line) — used to anchor reco #3 to actual pain

## Recommendation engine (3 ranked outputs)

Each reco emits this shape:

```
RECOMMENDATION 1 of 3
Action: <imperative, one line>
Why this is #1 for you: <which intake answer triggered it>
Conditional impact: <savings figure WITH the condition that makes it real>
Source: <specific Anthropic doc URL, evidential tier marked>
Do this today: <30-second concrete step>
```

Decision tree (priority order, first match wins):

| Trigger from intake | Reco | Conditional impact |
|---|---|---|
| Q1=Opus AND Q2>=30msgs | Switch default to Sonnet for coding, Opus only for architecture | "Sonnet is ~40% cheaper input than Opus, applies to all calls. [Anthropic pricing, vendor source]" |
| Q3=leave-on-always AND Q1!=Opus | Disable extended thinking for non-complex tasks | "Thinking tokens bill as output. Disabling for trivial work removes ~4-5x output overhead PER CALL where it was unnecessary; impact depends on what fraction of your calls are trivial. [Anthropic extended thinking docs, vendor source]" |
| Q2>=80 OR Q2=never-/clear | Adopt /clear discipline between tasks | "Each retained message reprocesses prior context. Past msg #50, per-message input cost grows ~linearly. No published % savings; behavioral change. [Anthropic best practices, vendor source]" |
| Q4>=500KB or repeated codebase | Move to subagents for codebase exploration | "Subagent context isolated; main thread receives summary only. Effective context gain depends on exploration intensity. [Anthropic best practices, vendor source]" |
| Q5 contains servers user can't justify | Disable unused MCP servers | "Each loaded MCP adds tool definition tokens per message. Magnitude depends on definitions size. [Anthropic costs doc, vendor source]" |
| Q1=I-don't-pick-explicitly | Pin model defaults | "Without explicit pick, you may be paying Opus rates by default depending on your plan. [Anthropic pricing, vendor source]" |
| Q6 = repeated multi-step task | Convert to a skill or saved prompt template | "Skills load on-demand and are cached as system content. No published % figure; structural improvement. [Anthropic skills docs, vendor source]" |

Fallback if intake too sparse: emit generic top-3 (caching, /clear, model routing) with explicit "this is a generic recommendation because intake was too sparse" disclaimer.

**Honesty rules** (Kimi-derived):
- Every percentage claim must include the condition under which it holds.
- "Causes" replaced by "associated with" everywhere.
- Sources tagged by tier: `[vendor]`, `[community-single-source]`, `[community-replicated]`, `[independent-research]`. Any reco whose primary citation is `[community-single-source]` is downgraded or dropped.
- The output ends with: "These are heuristics from a self-report intake, not analysis of your actual usage. For real per-user analysis, ask alex about the org telemetry dashboard."

## Distribution

1. Repo at `pago/claude-optimizer-skill` (private, Pago org).
2. SKILL.md + intake.md + recommendations.md + sources.md + plugin manifest for Cowork.
3. Org-shared in Settings > Skills > "Share with organization" — Team plan supports this.
4. Announce in #engineering Slack with a 2-paragraph note: what the skill does, what the telemetry collector does, the pseudonymization caveat, opt-out instructions for the OTel side (admin can exempt a user by hash).

## Falsification clauses (Kimi-derived)

- The "Sonnet is 40% cheaper than Opus" reco is falsified if Anthropic changes pricing such that the gap inverts. Audit pricing claims quarterly.
- The "v2.1.x cache bug" reco is from a single news source. Demoted to a footnote, not a top-3 reco, until corroborating evidence appears.
- The "2-4x velocity uplift" claim is removed entirely. We do not claim velocity outcomes — only the conditional unit economics of specific configurations.

## Open questions for codex

1. Does the intake-only design produce non-garbage recos in practice? Specifically: is 7 questions enough, or does the decision tree need 12-15 to avoid degenerate cases?
2. Should the skill split by surface — a chat-targeted variant vs. a Cowork-targeted variant — to avoid "you don't have hooks anyway, ignore the hooks reco" awkwardness?
3. Is the conditional-impact framing usable, or does it strip so much certainty that the recos feel toothless? Is there a middle ground?
4. Re Office/Cowork plugin packaging vs. SKILL.md drop-in: one artifact or two?
5. Is the OTel collector ingestion filter design (HMAC + drop) the right place for anonymization, or should we do it client-side (impossible for OTel) or at the dashboard query layer?
