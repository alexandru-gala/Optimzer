# Pago Optimizer

Internal Pago skill that gives colleagues 3 ranked recommendations for slowing their Claude usage-window burn. Ships in two surface variants (claude.ai chat, Office/Cowork), shares a single recommendation library, and pairs with an org-level OpenTelemetry collector for aggregate dashboards.

> Status: design + scaffold complete (v3.1). Awaiting decisions from @alexandru-gala on D1–D5 in `SYNTHESIS.md`, then soft-launch with 5 volunteers.

## Architecture

```
╔════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
║ DIAGRAM 1 — pago-optimizer architecture                                                               ║
╚════════════════════════════════════════════════════════════════════════════════════════════════════════════╝

┌──────────────────────────── pago-optimizer repo ────────────────────────────┐
│                                                                              │
│  ┌───────────────────────────────┐        ┌───────────────────────────────┐  │
│  │ chat/SKILL.md                 │        │ cowork/SKILL.md + plugin.json │  │
│  │ claude.ai chat surface        │        │ Office/Cowork surface         │  │
│  └──────────────┬────────────────┘        └────────────────┬──────────────┘  │
│                 ┆ imports                                   ┆ imports         │
│                 ┆                                           ┆                 │
│                 ▼                                           ▼                 │
│        ┌────────────────────────────────────────────────────────────┐         │
│        │ SHARED LIBRARY                                             │         │
│        │ recommendations.md: R1-R8, scored ranking                  │         │
│        │ sources.md: citation tiers                                │         │
│        └────────────────────────────────────────────────────────────┘         │
│                  ▲ only rule-change path to both surfaces                     │
└──────────────────┼───────────────────────────────────────────────────────────┘
                   ┆ shared/imports
                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ Distribution                                                                │
│ alex/org admin ──> Anthropic Settings ──> Skills ──> "Share with organization"│
│ Team plan share makes both skill variants available to the org               │
└──────────────────────────────────────────────────────────────────────────────┘


       Skill delivery path is separate from telemetry path:  Skill ≠ telemetry
       chat/SKILL.md has NO telemetry path; only Office/Cowork agents emit OTLP.

┌───────────────────── one-time telemetry config, passive after setup ─────────────────────┐
│ alex ──> Anthropic Settings ──> Monitoring panel                                         │
│          paste OTLP endpoint URL + Bearer token once                                    │
└───────────────────────────────────────┬──────────────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────── Cowork-only telemetry transport ─────────────────────────────┐
│ Office/Cowork agents ──OpenTelemetry──> Cloudflare Access (auth) ──> Pago otel-collector │
└──────────────────────────────────────────────────────────────────────────┬───────────────┘
                                                                           │ INGEST
                                                                           ▼
┌─────────────────────────────── Pago otel-collector ──────────────────────────────────────┐
│ ANONYMIZES AT INGEST, not query time                                                     │
│ drops: user.email, user.id, prompt.text, prompt.id, file.path, file.contents, tool.arguments│
│ keeps: model, tool_name, mcp_server, input/output/cache tokens, duration, day-bucket      │
│ keeps: session-length-bucket                                                             │
│ v0: NO per-user pseudonyms; aggregate-only by design                                     │
└──────────────────────────────────────────────┬───────────────────────────────────────────┘
                                               │ writes daily rows
                                               ▼
                              ┌────────────────────────────────┐
                              │ Postgres: org_aggregate_daily  │
                              └───────────────┬────────────────┘
                                              │ reads
                                              ▼
┌──────────────────────────────── alex-only dashboard ─────────────────────────────────────┐
│ 6 metrics: model mix, token totals, cache ratio, MCP load/invoke, tool counts,           │
│ session length distribution                                                             │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

## User flow (chat variant)

```
╔════════════════════════════════════════════════════════════════════════════════════════════╗
║ DIAGRAM 2 — chat variant user audit flow                                                 ║
╚════════════════════════════════════════════════════════════════════════════════════════════╝

┌──────────────────────────────── chat/SKILL.md on claude.ai chat surface ───────────────────────────────┐
│ User reads their own chat history; skill asks anchored questions, scores answers, returns recommendations│
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│ STEP 1: Past-chat RETRIEVAL (not scroll)   │
│ Skill asks the host Claude model to use    │
│ the "reference past chats" feature to      │
│ look up the user's last ~10 conversations. │
└──────────────┬─────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ Model retrieves and computes 3 counts:                                                   │
│ R-H1: sessions past 30 msgs?           R-H2: sessions with >50KB attachments?            │
│ R-H3: sessions where user explicitly picked the model?                                   │
│                                                                                          │
│ Reports ONE state honestly:                                                              │
│ • RETRIEVAL_OK       (≥5 sessions referenced)  → multiplier 2.0×                         │
│ • RETRIEVAL_PARTIAL  (<5 sessions / partial)   → multiplier 1.5×                         │
│ • RETRIEVAL_UNAVAILABLE (feature off / empty)  → fall back to user sidebar (mult 1.5×)   │
│                                                  or gut-feel only (mult 1.0×) if skipped │
│ NO fabrication — empty retrieval triggers fallback, not invented numbers.                │
└──────────────┬───────────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────┐
│ STEP 2: Habit questions    │
│ 6 forced-choice prompts    │
└──────────────┬─────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ I1 default model  │ I2 thinking default │ I3 MCP count │ I4 artifact type               │
│ I5 tool-loop frequency      │ I6 frustration moment                                        │
└──────────────┬───────────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────┐
│ STEP 3: Optional opener    │
│ User pastes recent chat    │
│ opener.                    │
└──────────────┬─────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ Skill scans opener for: length, attachments, vague framing, model mentions               │
└──────────────┬───────────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────── scoring ─────────────────────────────────────────────────┐
│ retrieval × 2.0  │  sidebar-fallback × 1.5  │  opener-match × 1.3  │  gut-feel × 1.0     │
│ Shared recommendations.md ranks matching rules R1-R8; sources.md supplies citation tier. │
└──────────────┬───────────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────── output ──────────────────────────────────────────────────┐
│ 3 ranked recommendations, each with:                                                     │
│ Impact: h/m/l                                                                            │
│ Confidence: vendor-pricing / telemetry / self-report                                     │
│ Anchor: which step triggered it                                                          │
│ Why                                                                                      │
│ Today action                                                                             │
│ Source URL                                                                               │
└──────────────┬───────────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────── end ─────────────────────────────────────────────────────┐
│ Honest disclaimer: "I retrieved what I could, scored it, and asked you the rest"         │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

> Diagrams produced by Codex (codex CLI 0.125.0) on 2026-05-01.

## What this repo contains

```
.
├── README.md              ← you are here
├── DESIGN.md              ← v2 architecture (skill + telemetry + distribution)
├── SYNTHESIS.md           ← Codex + Kimi merged review, 5 open decisions for alex
├── COLLECTOR.md           ← v2 OTel collector setup (superseded by v3)
├── skill/SKILL.md         ← v2 skill scaffold (single-artifact, superseded by v3)
└── v3/                    ← current target
    ├── CHANGELOG.md       ← v2 → v3 diff with rollback plan
    ├── recommendations.md ← shared library, 8 rules, scored ranking
    ├── sources.md         ← citation tiers + quarterly audit checklist
    ├── COLLECTOR.md       ← v0 collector (org-aggregate-only, no per-user pseudonyms)
    ├── chat/SKILL.md      ← claude.ai chat variant
    └── cowork/
        ├── SKILL.md       ← Office/Cowork variant
        └── plugin.json    ← Cowork plugin manifest
```

## What the skill does

Runs a 9-question structured intake (model default, /clear discipline, thinking default, attachment size, MCP count, artifact type, tool-loop frequency, model-override habit, free-text frustration), scores all 8 recommendation rules, returns the top 3 with `Impact`, `Confidence`, `Why`, and a 30-second action.

The skill does **not** transmit anything externally. It is a heuristic checklist. Real per-user analytics comes from a separate org-admin collector layer.

## What the telemetry does

Independent of the skill. Org admin (alex) pastes a Pago-controlled OTLP endpoint into Anthropic's `Settings → Monitoring` panel; Office/Cowork agent events stream to a Pago `otel-collector` that anonymizes at ingest (drops user.email, prompt.text, etc.; keeps model, token counts, MCP load/invoke ratios, session length buckets) and writes only daily org-aggregate rows to Postgres. claude.ai chat events are not in this stream — Anthropic doesn't expose chat OTel.

## How it was built

1. Three parallel research agents grounded the design in current Anthropic docs (Claude Skills, Cowork, Hooks, Usage API, OTel surface).
2. Kimi (epistemic adversarial review) flagged 5 honesty problems with the v1 framing — fixed in v2.
3. Codex (technical adversarial review) flagged 5 structural problems with v2 — fixed in v3.
4. v3 ships under its own subdirectory so v2 stays intact for rollback.

Full review trail: `SYNTHESIS.md`.

## Next steps

See `v3/CHANGELOG.md` "What still needs alex" section.

## License

Pago internal — proprietary.
