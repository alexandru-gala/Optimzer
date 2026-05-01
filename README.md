# Pago Optimizer

Internal Pago skill that gives colleagues 3 ranked recommendations for slowing their Claude usage-window burn. Ships in two surface variants (claude.ai chat, Office/Cowork), shares a single recommendation library, and pairs with an org-level OpenTelemetry collector for aggregate dashboards.

> Status: design + scaffold complete (v3). Awaiting decisions from @alexandru-gala on D1–D5 in `SYNTHESIS.md`, then soft-launch with 5 volunteers.

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
