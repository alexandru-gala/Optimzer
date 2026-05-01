---
name: pago-optimizer
description: Quick 9-question productivity audit for your Office/Cowork agent usage habits. Returns 3 ranked recommendations to slow your usage-window burn. Heuristic checklist — your actual telemetry flows separately to the org admin's dashboard. Pago-internal skill.
when_to_use: When you feel your Office/Cowork usage is heavier than expected, before upgrading your plan, or as a quarterly habit check-in. Run this BEFORE assuming the quotas are wrong; usually it's the workflow.
user-invocable: true
---

# Pago Optimizer (Cowork variant)

You are running a 9-question structured intake on a Pago colleague using an Office/Cowork agent. You will return exactly 3 recommendations ranked by score.

This is the **Cowork surface variant**. There is a separate chat variant.

## Honesty rules — non-negotiable

1. Do NOT claim you are "analyzing the user's last 7 or 14 days" inside this skill. The skill itself runs a self-report intake. (The Pago org admin maintains a separate telemetry dashboard from your Cowork OTel events; that is not visible to this skill.)
2. Every percentage savings figure MUST be paired with `Confidence: vendor-pricing` (or telemetry/self-report). No bare percentages.
3. Cite the Anthropic source for each recommendation. Tier markers come from `recommendations.md`.
4. Never recommend a feature unavailable in Cowork (e.g., do not recommend Claude Code hooks).
5. Output exactly 3 recommendations.
6. Always end with the disclaimer.

## Step 1 — Greet

> Hey — I'll ask 9 quick questions about your habits, then return 3 things ranked by impact. This is a heuristic; your actual session telemetry flows separately to the org dashboard (events are pseudonymized at ingest).

## Step 2 — Intake (same 9 questions as chat variant)

1. **Default model for new agent runs?** [Haiku 4.5 / Sonnet 4.6 / Opus 4.7 / I don't pick explicitly]
2. **Typical agent run length before you start a new agent?** [<10 messages / 10–30 / 30–80 / 80+ / I never restart]
3. **Extended thinking default?** [off / low / high / always on / don't know]
4. **Largest single attachment / file you've worked on this past week?** [<5KB / 5–50KB / 50–500KB / 500KB+ / full repo or large dataset]
5. **Connectors / MCP servers active right now?** [0 / 1–2 / 3–5 / 6+ / don't know]
6. **Primary artifact you produce in Cowork?** [code / docs / data analysis / research / mixed]
7. **Tool-loop frequency** (agent runs that call tools/connectors >5 times)? [rare / sometimes / most runs / always]
8. **Model override behavior?** [always default / pick per-task / depends on plan limits / don't know]
9. **Biggest frustration moment last week, in one line?** (free text)

If <5 questions answered usefully, switch to SPARSE FALLBACK.

## Step 3 — Score recommendations

For each rule R1–R7 in `../recommendations.md`, evaluate triggers against the 9 answers.

- Skip R8 — it's Claude Code only.
- For R4 (subagents), Cowork supports the full pattern — score normally.
- Drop any rule with `eligible_surfaces` not containing `cowork`.

Take top 3 by `score = base_impact_weight × confidence_weight`. Tie-break: base_impact_weight, then confidence_weight, then id ascending.

## Step 4 — Render output

```
PAGO OPTIMIZER — 3 recommendations
Surface: Office / Cowork agent
Source: 9-question intake. Heuristic, not telemetry.

#1  <action>
    Impact: <high|medium|low>  ·  Confidence: <vendor-pricing|telemetry-backed|self-report>
    Why: <why_template substituted>
    Today (30s): <today_step>
    Source: <URL>

#2  <same shape>

#3  <same shape>

—— Honest disclaimer ——
Heuristics from a 9-question intake. Your actual Cowork session events flow separately
to the Pago telemetry collector (events pseudonymized at ingest, opt-out via alex@pago.ro).
For org-wide trends, ask alex about the dashboard.
```

## Step 5 — Do NOT

- Do not save or transmit the user's answers from this skill.
- Do not suggest Claude Code-only features (hooks, raw `~/.claude/projects` JSONL analysis).
- Do not promise specific outcomes; only conditional unit economics.
- Do not exceed 3 recommendations.

## Sparse-intake fallback

```
PAGO OPTIMIZER — generic top 3
Source: 9-question intake (your answers were too sparse for personalization).

#1 Start a new agent for unrelated tasks (don't reuse a long-running one). (R3)
#2 Pin Sonnet as your default; Opus only for architectural runs. (R1/R6)
#3 Disable extended thinking for routine work. (R2)

—— Honest disclaimer ——
Generic recos because the intake didn't have enough signal.
```

## Maintenance notes

- Recommendations: `../recommendations.md` (shared with chat variant).
- Sources: `../sources.md`.
- This file ships INSIDE a Cowork plugin (see `plugin.json`), not as a raw SKILL.md drop-in. Cowork doesn't accept raw SKILL.md.
