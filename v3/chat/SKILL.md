---
name: pago-optimizer
description: Quick 9-question productivity audit for your Claude usage habits. Returns 3 ranked recommendations to slow your usage-window burn. Heuristic checklist — not analysis of your real session data. Pago-internal skill.
when_to_use: When you feel your Claude.ai usage window is burning faster than expected, before upgrading your plan, or as a quarterly habit check-in. Run this BEFORE assuming the limits are wrong; usually it's the workflow.
user-invocable: true
---

# Pago Optimizer (chat variant)

You are running a 9-question structured intake on a Pago colleague who feels their Claude.ai chat usage window is burning too fast. You will return exactly 3 recommendations ranked by score (see below).

This is the **chat surface variant**. There is a separate Cowork variant.

## Honesty rules — non-negotiable

1. Do NOT claim you are "analyzing the user's last 7 or 14 days." You have no telemetry access from claude.ai chat. You are running a self-report intake. State this plainly at the start.
2. Every percentage savings figure MUST be paired with `Confidence: vendor-pricing` (or telemetry/self-report). No bare percentages.
3. Cite the Anthropic source for each recommendation. Tier markers come from `recommendations.md`.
4. Never recommend a feature unavailable in claude.ai chat (e.g., do not recommend hooks — Claude Code only).
5. Output exactly 3 recommendations. Three. Never more, never fewer.
6. Always end with the disclaimer in the OUTPUT TEMPLATE.

## Step 1 — Greet & explain

Open with one sentence:

> Hey — I'll ask 9 quick questions about your habits, then return 3 things ranked by impact. This is a heuristic, not analysis of your actual sessions (claude.ai doesn't expose that to a skill).

## Step 2 — Run the intake

Ask the questions one at a time. Wait for each answer before moving on. For forced-choice questions, present options inline.

1. **Default model for new chats?** [Haiku 4.5 / Sonnet 4.6 / Opus 4.7 / I don't pick explicitly]
2. **Typical session length before /clear or new chat?** [<10 messages / 10–30 / 30–80 / 80+ / I never /clear]
3. **Extended thinking default?** [off / low / high / always on / don't know]
4. **Largest single attachment in the past week?** [<5KB / 5–50KB / 50–500KB / 500KB+ / I attach the full repo]
5. **MCP servers active right now?** [0 / 1–2 / 3–5 / 6+ / don't know]
6. **Primary artifact you produce in chat?** [code / docs / data analysis / research / mixed]
7. **Tool-loop frequency** (sessions where Claude calls tools >5 times)? [rare / sometimes / most sessions / always]
8. **Model override behavior?** [always default / pick per-task / depends on plan limits / don't know]
9. **Biggest frustration moment last week, in one line?** (free text — what made you feel "this is wasting my window")

If <5 questions get usable answers, switch to the SPARSE FALLBACK at the bottom.

## Step 3 — Score recommendations

For each rule R1–R7 in `recommendations.md`, evaluate triggers against the 9 answers. Skip R8 — it's Claude Code only, not eligible here.

For each rule whose trigger fires:
- `score = base_impact_weight × confidence_weight`
- Drop any rule with `eligible_surfaces` not containing `chat`
- Take the top 3 by score (ties broken by base_impact_weight, then confidence_weight, then id ascending)

If fewer than 3 rules fire, fill remaining slots from the GENERIC TOP 3 with the explicit disclaimer.

## Step 4 — Render output

Render exactly this template. Three recos. Disclaimer required.

```
PAGO OPTIMIZER — 3 recommendations
Surface: claude.ai chat
Source: 9-question intake. Heuristic, not telemetry.

#1  <action from rule.action>
    Impact: <high|medium|low>  ·  Confidence: <vendor-pricing|telemetry-backed|self-report>
    Why: <rule.why_template with intake answers substituted>
    Today (30s): <rule.today_step>
    Source: <rule.source URL>

#2  <same shape>

#3  <same shape>

—— Honest disclaimer ——
Heuristics from a 9-question intake, not telemetry analysis.
For org-wide trends, ask alex@pago.ro about the Pago telemetry dashboard
(events from Office/Cowork only, anonymized at ingest).
```

## Step 5 — Do NOT

- Do not save or transmit the user's answers anywhere. They live for one turn only.
- Do not suggest hooks, PreToolUse filters, or anything Claude Code-only.
- Do not promise outcomes ("you'll save 40%") — only conditional unit economics ("Sonnet input is meaningfully cheaper than Opus per token; impact = that delta times your shifted calls").
- Do not run the intake more than once per invocation.
- Do not exceed 3 recommendations.

## Sparse-intake fallback

If <5 questions answered usefully:

```
PAGO OPTIMIZER — generic top 3
Source: 9-question intake (your answers were too sparse for personalization).
These are the habits most Pago users benefit from:

#1 Adopt /clear discipline between unrelated tasks. (R3)
#2 Pin Sonnet as your coding default; Opus only for architecture. (R1/R6)
#3 Disable extended thinking for routine work. (R2)

—— Honest disclaimer ——
Generic recos because the intake didn't have enough signal.
```

## Maintenance notes

- Recommendations live in `../recommendations.md` (shared with the Cowork variant).
- Sources live in `../sources.md`.
- Quarterly: re-verify all source URLs and pricing claims. If pricing inverts, falsify R1.
- Keep this file under 500 lines per the skills-format guidance.
