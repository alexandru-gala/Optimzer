# Pago Optimizer — Codex + Kimi Review Synthesis

**Date**: 2026-05-01
**Inputs**: DESIGN.md v2, Kimi epistemic review, Codex technical review
**Owner**: alex@pago.ro
**Verdict (joint)**: **Ship after fixes**. Core idea sound. v2 fixed honesty issues but is still overbuilt on telemetry and underbuilt on intake quality.

---

## What both reviewers agreed on

1. **One artifact serving three surfaces is wrong.** Codex: split into chat `SKILL.md` + Cowork plugin, share an internal recommendation library. Kimi was silent here but flagged "horoscope dressed as analytics" if the chat surface conflates with Code's real-data capability.
2. **The 7-question intake is the weakest part of v2.** Codex: replace Q5–Q7 (free text) with higher-signal questions, target 9–10 not 7. Kimi: a 7-question self-report cannot legitimately back any "personalization" claim and currently undersells confidence anyway.
3. **The recommendation ranking is wrong by construction.** First-match-wins fakes precision. Score all triggers, emit `Impact` (high/med/low) + `Confidence` (vendor-pricing / telemetry / self-report) labels, then rank.
4. **Telemetry layer is overbuilt for v0.** Codex: drop the dashboard initially; store daily org-level aggregates only; do not retain per-user pseudonyms until a concrete intervention requires them. Kimi: pseudonymization framing is honest but the side channel is the privacy weakest link, narrow it.
5. **Conditional-impact framing is overcorrected.** Codex: replace "Sonnet is 40% cheaper input applied to..." legalese with `Impact: high · Confidence: vendor-pricing-backed · Why: ...`. Kimi accepted v2's caveats but warned the user will ignore courtroom-style hedges.

## Where they diverged

- **Kimi** wanted MORE epistemic hedging. **Codex** wanted decisive recommendations with structured uncertainty (Impact + Confidence labels), not legalistic prose. The synthesis: keep Kimi's substance (no unconditional %), but apply Codex's *form* (labels, not paragraphs).
- **Kimi** flagged the 50-user re-ID risk on pseudonyms. **Codex** went further: drop per-user pseudonyms entirely until a concrete need exists. Synthesis: ship v0 without per-user pseudonyms; only daily org-aggregate rows; revisit if/when an intervention design needs them.

---

## v3 corrections (to apply before shipping)

### Mechanical corrections (apply now, no decisions needed)

| # | Where | Change |
|---|---|---|
| C1 | SKILL.md output template | Replace verbose "Conditional impact: <para>" with two labels + one sentence: `Impact: <high/med/low>`, `Confidence: <vendor-pricing / telemetry / self-report>`, `Why: <one sentence>` |
| C2 | DESIGN.md "Decision tree" section | Change from first-match-wins to scored ranking. Each rule emits a score = base_impact × confidence × eligibility. Top 3 by score, not by trigger order. |
| C3 | DESIGN.md Layer 2 + COLLECTOR.md | Drop per-user pseudonym storage from v0. Store daily org-level aggregates only: counts/sums grouped by `model`, `tool_name`, `mcp_server`, `surface`. No `user_pseudonym` column in v0 schema. |
| C4 | DESIGN.md Layer 2 | Define the exact 5–6 metrics the dashboard surfaces BEFORE deciding what fields to retain. Codex's exact phrasing: "do not design the collector around vague future dashboard ideas." |
| C5 | COLLECTOR.md | Add operational hardening: collector access is admin-only (1Password-gated SSH only), no raw debug logging (otelcol log level=warn in prod), no failed-payload retention >24h, no staging mirror with full fidelity, automated test that proves dropped fields stay dropped on every config change. |
| C6 | SKILL.md | Drop the line "Optionally accepts a pasted summary from the claude.ai data export." Codex: this is awkward UX nobody will use. |
| C7 | SKILL.md eligibility matrix | Make surface-specific copy by FORK in the SKILL.md, not by IF/ELSE inside one prompt. Two SKILL.md variants drawing from a shared `recommendations.md`. |

### Decisions for alex (need a call)

| # | Decision | Codex says | Kimi says | Recommendation |
|---|---|---|---|---|
| D1 | Question count: 7 vs 9-10? | 9–10 sharply chosen, not 12–15 | 7 is fine if framing is honest | **Go with 9** — replace v2's Q5/Q6/Q7 free-text with structured Q5 (MCP server count), Q6 (artifact type), Q7 (tool-loop frequency), Q8 (pain point free-text, kept short), Q9 (model override habit). |
| D2 | One artifact or two? | Two (chat SKILL.md + Cowork plugin) | n/a | **Two**, with a shared `recommendations.md` source library. Codex is right; the surface compat matrix in v2 leaks awkwardness. |
| D3 | Per-user pseudonyms day-1? | No, daily org-aggregate only | Pseudo with k≥5 cohorts | **No** — go with daily org-aggregates for v0. Add per-user tracking only when an intervention needs A/B-style attribution, with a privacy review at that point. |
| D4 | Ship Claude Code variant too? | Optional 3rd entrypoint | n/a | **Defer** — focus on chat + Cowork (the surfaces colleagues actually use). Code variant can come later if useful, but it's a different product (real telemetry from JSONL changes the framing). |
| D5 | "v2.1.x cache bug" reco placement? | Not a top reco | Single-source citation, demote | **Drop entirely from skill output**. Keep in `sources.md` as a footnote. Don't surface to users. |

---

## v3 question set (proposed, replaces v2's 7)

1. **Default model** [Haiku / Sonnet / Opus / no explicit pick]
2. **Session length before /clear** [<10 / 10–30 / 30–80 / 80+ / never]
3. **Extended thinking default** [off / low / high / always on / don't know]
4. **Largest single attachment, last week** [<5KB / 5–50KB / 50–500KB / 500KB+ / full repo]
5. **MCP servers active** [0 / 1–2 / 3–5 / 6+ / don't know] *(structured, not free-text)*
6. **Primary artifact type produced** [code / docs / data analysis / research / mixed]
7. **Tool-loop frequency** (sessions where Claude calls tools >5 times) [rare / sometimes / most sessions / always]
8. **Model override behavior** [always default / pick per-task / depends on plan limits / don't know]
9. **Biggest frustration moment, last week** (one line, free text — kept for qualitative anchor of reco #3)

This swaps v2's two weakest free-text questions (Q6 repeated-task, Q7 frustration) for structured Q5–Q8 plus keeps one short free-text for color.

---

## v3 output template (proposed)

```
PAGO OPTIMIZER — 3 recommendations
Surface: <claude.ai chat | Office/Cowork>
Source: 9-question intake. Heuristic, not telemetry.

#1  Switch default to Sonnet for coding
    Impact: high · Confidence: vendor-pricing-backed
    Why: You said you default to Opus on long coding sessions. Sonnet is meaningfully cheaper per input token; for the same task profile this maps directly to lower spend.
    Today (30s): /model claude-sonnet-4-6 in your next chat. Reserve Opus for architecture sessions.

#2  Disable extended thinking for routine work
    Impact: medium · Confidence: vendor-pricing-backed
    Why: You said thinking is "always on" and your repeated work is <triage / formatting / quick edits>. Thinking tokens bill as output; disabling on routine calls removes that overhead.
    Today (30s): /effort low or off in routine sessions. Re-enable per-task only when reasoning depth matters.

#3  Adopt /clear discipline
    Impact: high · Confidence: self-report (no published %)
    Why: You said sessions go 80+ messages. Each retained message reprocesses prior context.
    Today (30s): /clear before each unrelated task. New task = new context.

—— Honest disclaimer ——
Heuristics from a 9-question intake, not telemetry analysis. For org-wide trends, ask alex@pago.ro about the Pago telemetry dashboard.
```

---

## What this does NOT change from v2

- Org-shared distribution via Settings → Skills → Share with organization. Still right.
- Cowork OTel telemetry path (the screenshot unlock). Still the cleanest org-side telemetry mechanism.
- HMAC + drop-PII at ingest as the anonymization layer. Still right (just narrower retention).
- Honesty rules (no unconditional %, source tier marking, falsification clauses). Still required.
- Dropping Tier B (bundled MCP telemetry connector for chat users). Still dropped.

---

## Recommended next steps (priority order)

1. **Lock D1–D5 with alex.** 10-minute decision call — these change what gets built.
2. **Apply C1–C7 mechanical corrections** to DESIGN.md, SKILL.md, COLLECTOR.md.
3. **Fork SKILL.md into chat + Cowork variants** with a shared `recommendations.md`.
4. **Stand up the OTel collector** with v0 schema (org-aggregate only, no per-user rows).
5. **Soft-launch with 5 Pago volunteers** for one week. Measure: completion rate of the intake, quality of the 3 recos (each volunteer rates 1–5), self-reported burn change.
6. **Iterate** based on the 5-volunteer feedback before org-wide rollout.

## Audit trail

- Kimi review: kimi session `d6bffb8d-a8b6-4510-8ae2-0aea75c4fbd1`
- Codex review: `/tmp/codex-review.out` (committed-locally), exec on 2026-05-01
- Research digests: archived in agents `a3ef7f2d9d20a917f`, `acb265a55212aece1`, `a6b2ac093be4f0682`
