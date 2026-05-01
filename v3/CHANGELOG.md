# Pago Optimizer — Changelog

## v3.1 — 2026-05-01 (chat variant: history-anchored calibration)

**Driver**: alex feedback after reading the README — chat variant should "check into existing past chats complementary to the questions."

**Reality check**: claude.ai chat skills cannot programmatically read past conversations. There is no API surface. Anthropic's only path is the human-initiated data export (Settings → Privacy → Export Data → email zip).

**What landed instead**: a 3-step flow in the chat variant.

- **Step 1 (new)**: User opens chat history sidebar; skill asks 3 anchored questions (H1–H3) about their last ~10 sessions — sessions past 30 messages, sessions with large attachments, sessions where they explicitly picked a model. These map directly to R3, R4, R6.
- **Step 2 (revised)**: Habit intake reduced from 9 → 6 questions (I1–I6). The 3 questions removed (`session length`, `attachment size`, `model override behavior`) are now better-answered by the H-anchored Step 1.
- **Step 3 (new, optional)**: User can paste a recent chat opener for an artifact-level signal. Skill scans for length, attachments, vague framing, model mentions. Skip is fine.
- **Scoring (revised)**: Added `calibration_multiplier` to the scored ranking formula. H-anchored triggers weight 1.5×; opener-pattern triggers weight 1.3×; gut-feel triggers (I1–I6) stay at 1.0×. If user skips history (`H_SKIPPED=true`), all H multipliers fall back to 1.0.
- **Honesty**: New rule #7 in the skill — past-chat answers from Step 2 carry **higher confidence weight** in scoring than gut-feel answers from Step 3. Output template's `Source` line now reflects what was actually used (history-anchored + intake + opener / history-anchored + intake / intake only).

**Cowork variant**: unchanged. Cowork users have a separate telemetry path (the OTLP collector) that produces real per-session data org-wide — no need for an in-skill anchored-question flow.

**Files touched**:
- `v3/chat/SKILL.md` — restructured into 3 steps
- `v3/recommendations.md` — R3, R4, R6 triggers updated to accept H-anchors; scored ranking section updated with calibration_multiplier

**Rollback**: `git revert <commit>` — single-commit change.

---

## v3.0 — 2026-05-01 (initial fork from v2)

**Date**: 2026-05-01
**Trigger**: Codex + Kimi adversarial reviews of v2.

## Summary

v3 implements Codex's "ship after fixes" verdict. v2's honesty rules (Kimi-derived) are preserved unchanged; v2's structural decisions (one artifact, first-match-wins, per-user pseudonyms) are tightened.

## Changes

### Mechanical (no decision needed — auto-applied per SYNTHESIS.md C1–C7)

| ID | What changed |
|---|---|
| C1 | Output template: replaced verbose "Conditional impact" paragraph with `Impact: <h/m/l> · Confidence: <vendor-pricing/telemetry/self-report>` + one-sentence Why. |
| C2 | Recommendation ranking: switched from first-match-wins to scored ranking (`base_impact_weight × confidence_weight × eligibility`). |
| C3 | Collector schema: dropped per-user pseudonym storage. v0 stores only `org_aggregate_daily` rows. |
| C4 | Collector design: defined the 6 dashboard metrics explicitly before specifying retained fields. |
| C5 | Collector hardening: admin-only access, prod log level=warn, 24h max failed-payload retention, no full-fidelity staging mirror, CI drop-test on every config change. |
| C6 | Removed the "paste your claude.ai data export" path from the skill — awkward UX nobody uses. |
| C7 | Forked the skill: `v3/chat/SKILL.md` and `v3/cowork/SKILL.md` import from shared `v3/recommendations.md`. |

### Decisions applied (alex's call — recommended picks from SYNTHESIS.md)

| ID | Decision | What landed |
|---|---|---|
| D1 | 9 questions, not 7 | New Q5 (MCP count, structured), Q6 (artifact type), Q7 (tool-loop frequency), Q8 (model override habit). Free-text Q9 kept for qualitative anchor. |
| D2 | Two artifacts, shared library | `v3/chat/` and `v3/cowork/` each ship their own SKILL.md, both reading `v3/recommendations.md`. Cowork variant additionally ships `plugin.json`. |
| D3 | No per-user pseudonyms in v0 | Collector stores org-aggregate rows only. v0 → v1 promotion criteria documented in `v3/COLLECTOR.md`. |
| D4 | No Claude Code variant in v0 | Pago colleagues use chat + Cowork. Code variant deferred. R8 (CLAUDE.md → skills) remains in `recommendations.md` but is filtered out of both shipped variants. |
| D5 | "v2.1.x cache bug" reco dropped from skill output | Demoted to `sources.md` footnote. Re-evaluate quarterly if independently corroborated. |

### Preserved from v2 (explicit non-changes)

- Honesty rules (Kimi-derived): no "7d/14d analysis" claim, no unconditional %, "pseudonymized" not "anonymous", citation tier marking, falsification clauses, "2-4× velocity uplift" claim NOT made.
- Distribution mechanism: org-shared via Settings → Skills → Share with organization (Team plan).
- Telemetry transport: Cowork OTel push to Pago-controlled OTLP endpoint, configured in Anthropic Settings → Monitoring (per the screenshot from alex).
- Anonymization layer: ingest, not query-time.
- Tier B (bundled MCP telemetry connector for chat users): still dropped.

## File map

```
v3/
├── recommendations.md     ← shared library (8 rules R1-R8, scored)
├── sources.md             ← citation tiers + audit checklist
├── COLLECTOR.md           ← v0 collector setup, hardening, queries
├── CHANGELOG.md           ← this file
├── chat/
│   └── SKILL.md           ← claude.ai chat variant, 9Q intake
└── cowork/
    ├── SKILL.md           ← Cowork variant, 9Q intake
    └── plugin.json        ← Cowork plugin manifest
```

## What still needs alex

1. **Read & approve / correct the 5 D-decisions.** I picked recommended defaults; if you want different, tell me and I'll re-cut.
2. **Decide soft-launch cohort.** I suggested 5 Pago volunteers for one week. Pick the 5.
3. **Stand up the OTel collector.** Choice between Fly.io ($10/mo) vs. existing Pago infra. `COLLECTOR.md` has the config.
4. **Configure the Anthropic Settings → Monitoring panel** with the collector URL + bearer token once collector is live.
5. **Org-share the chat skill** via Settings → Skills → Share with organization. Cowork plugin needs to be installed via the plugin install flow (verify what that looks like in your tenant).

## Roll-back plan

v3 lives under `v3/` exclusively. `DESIGN.md`, `COLLECTOR.md` (root), `skill/SKILL.md` (root) all retain v2 unchanged. To revert to v2: delete `v3/`. To proceed with v3: archive v2 files into a `v2-archive/` directory and promote `v3/` content to top-level.
