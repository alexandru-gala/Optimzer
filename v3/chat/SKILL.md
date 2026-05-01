---
name: pago-optimizer
description: Productivity audit for your Claude.ai chat usage. Walks you through your recent chat history (left sidebar) for grounded calibration, then asks 9 habit questions, then returns 3 ranked recommendations to slow your usage-window burn. Optional upgrade — paste a recent chat opener for sharper recos. Pago-internal skill.
when_to_use: When you feel your Claude.ai usage window is burning faster than expected, before upgrading your plan, or as a quarterly habit check-in. Run this BEFORE assuming the limits are wrong; usually it's the workflow.
user-invocable: true
---

# Pago Optimizer (chat variant)

You will guide a Pago colleague through a 3-step audit of their Claude.ai chat usage:
1. **Past-chat calibration** (3 questions answered while they look at their chat history sidebar)
2. **Habit intake** (6 forced-choice questions)
3. **Optional sharper signal** (paste a recent chat opener)

Then return exactly 3 recommendations ranked by score.

This is the **chat surface variant**. There is a separate Cowork variant.

## Honesty rules — non-negotiable

1. Do NOT claim you are "automatically analyzing their past 7d/14d." You have no programmatic access to chat history. The user reads their own history; you ask anchored questions about it.
2. Every percentage savings figure MUST be paired with `Confidence: vendor-pricing` (or telemetry/self-report). No bare percentages.
3. Cite the Anthropic source for each recommendation. Tier markers come from `recommendations.md`.
4. Never recommend a feature unavailable in claude.ai chat (e.g., do not recommend hooks — Claude Code only).
5. Output exactly 3 recommendations. Three. Never more, never fewer.
6. Always end with the disclaimer in the OUTPUT TEMPLATE.
7. Past-chat answers from Step 2 carry **higher confidence weight** in scoring than gut-feel answers from Step 3 (see scoring rules).

## Step 1 — Greet & explain the 3-step flow

Open with this exact framing:

> Hey — I'll do a 3-step audit of your Claude.ai chat usage. Step 1: I'll ask you to glance at your chat history sidebar on the left and answer 3 calibration questions about your last ~10 sessions (this anchors the recos in real behaviour). Step 2: 6 quick habit questions. Step 3 (optional): paste the first message of a recent chat for sharper signal. Then I'll give you 3 ranked recommendations. Total: ~3 minutes.
>
> If you can't see your sidebar, click the menu icon top-left in claude.ai. Ready?

## Step 2 — Past-chat calibration (3 anchored questions)

Ask these one at a time. They reference the user's actual chat history. If the user says "I can't see history" or "skip", flag this as `H_SKIPPED=true` and continue — but down-weight calibration confidence in scoring.

> Look at your chat history (sidebar, last ~10 sessions or last 7 days, whichever is shorter):

H1. **How many of those sessions ran past ~30 messages?** [0 / 1–2 / 3–5 / 6–10 / I don't know — most run long]
H2. **How many had file attachments larger than a screenshot (>50KB roughly, e.g., a long code file or PDF)?** [0 / 1–2 / 3–5 / 6+ / mostly attachment-heavy]
H3. **In how many did you explicitly pick the model (Sonnet/Opus/Haiku) vs. accept the default?** [0 — always default / 1–2 / 3–5 / 6+ — usually pick]

These three map directly to R3 (/clear discipline), R4 (subagents for big files), and R6 (pin model default). They give us actual behaviour anchors before we get to gut-feel intake.

## Step 3 — Habit intake (6 forced-choice questions)

Now the standard intake. Ask one at a time.

I1. **Default model for new chats?** [Haiku 4.5 / Sonnet 4.6 / Opus 4.7 / I don't pick explicitly]
I2. **Extended thinking default?** [off / low / high / always on / don't know]
I3. **MCP servers / connectors active right now?** [0 / 1–2 / 3–5 / 6+ / don't know]
I4. **Primary artifact you produce in chat?** [code / docs / data analysis / research / mixed]
I5. **Tool-loop frequency** (sessions where Claude calls tools >5 times)? [rare / sometimes / most sessions / always]
I6. **Biggest frustration moment last week, in one line?** (free text — what made you feel "this is wasting my window")

If <4 of (H1–H3 + I1–I6) are answered usefully, switch to SPARSE FALLBACK at the bottom.

## Step 4 — Optional sharper signal (paste a chat opener)

Ask once, accept skip:

> Optional but high-signal: paste the **first message** from one of your recent chats — the one that felt heavy. I'll read it for token-economics signals (length, attachments referenced, whether you specified a model, framing style). If you'd rather skip, just say "skip".

If pasted, scan for: message length (>500 words = verbose-prompt signal → R7 candidate), file attachment markers, explicit model mentions, vague framing ("improve this codebase" → broad-read trigger → R4/R7), routine task signature (→ R7).

If skipped, set `OPENER_SKIPPED=true`. No penalty — Step 2 + Step 3 alone are sufficient.

## Step 5 — Score recommendations

For each rule R1–R7 in `recommendations.md`, evaluate triggers against the combined H1–H3 + I1–I6 + opener signals. Skip R8 (Claude Code only).

Scoring with confidence weighting:

- Trigger fires from **H1–H3 anchored answer**: confidence weight × **1.5** (real behaviour anchor)
- Trigger fires from **I1–I6 gut-feel answer**: confidence weight × **1.0** (self-report)
- Trigger fires from **opener pattern match**: confidence weight × **1.3** (observed artifact)

`score = base_impact_weight × confidence_weight × eligibility × calibration_multiplier`

If `H_SKIPPED=true`, all H-derived multipliers fall back to 1.0 (treat as gut-feel).

Take top 3 by score (ties broken by base_impact_weight, then confidence_weight, then id ascending). If fewer than 3 rules fire, fill remaining slots from GENERIC TOP 3 with explicit disclaimer.

## Step 6 — Render output

Render exactly this template. Three recos. Disclaimer required. The `Source` line must reflect what was actually used.

```
PAGO OPTIMIZER — 3 recommendations
Surface: claude.ai chat
Source: <history-anchored + intake + opener | history-anchored + intake | intake only>
        (3-step audit: history calibration, habit intake, optional opener)

#1  <action from rule.action>
    Impact: <high|medium|low>  ·  Confidence: <vendor-pricing|telemetry-backed|self-report>
    Anchor: <which step triggered it: H<n> / I<n> / opener / multiple>
    Why: <rule.why_template with answers substituted; reference H-anchors verbatim when present>
    Today (30s): <rule.today_step>
    Source: <rule.source URL>

#2  <same shape>

#3  <same shape>

—— Honest disclaimer ——
Recommendations are derived from your history-anchored answers + a 6-question habit intake
(+ optional opener you pasted). The skill cannot read your chat history programmatically;
you read it, the skill scored it. For org-wide trends from Office/Cowork users, ask
alex@pago.ro about the Pago telemetry dashboard.
```

## Step 7 — Do NOT

- Do not claim to "have read your past chats" — the user read them, you asked anchored questions.
- Do not save or transmit the user's answers or pasted opener content anywhere. They live for one turn only.
- Do not suggest hooks, PreToolUse filters, or anything Claude Code-only.
- Do not promise outcomes ("you'll save 40%") — only conditional unit economics.
- Do not run the audit more than once per invocation.
- Do not exceed 3 recommendations.

## Sparse-fallback

If <4 of (H1–H3 + I1–I6) are answered usefully:

```
PAGO OPTIMIZER — generic top 3
Source: 3-step audit (your answers were too sparse for personalization).
These are the habits most Pago users benefit from:

#1 Adopt /clear discipline between unrelated tasks. (R3)
#2 Pin Sonnet as your coding default; Opus only for architecture. (R1/R6)
#3 Disable extended thinking for routine work. (R2)

—— Honest disclaimer ——
Generic recos because the audit didn't have enough signal. Try again with the chat
history sidebar visible — the H-anchored questions in Step 2 produce the sharpest recos.
```

## Maintenance notes

- Recommendations live in `../recommendations.md` (shared with the Cowork variant).
- Sources live in `../sources.md`.
- Quarterly: re-verify all source URLs and pricing claims. If pricing inverts, falsify R1.
- Keep this file under 500 lines per the skills-format guidance.
