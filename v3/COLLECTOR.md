# Pago OTel Collector — v0 Setup

**Audience**: Pago admin (alex) configuring the OTLP endpoint that Anthropic Settings → Monitoring will point at for Office/Cowork agents.
**Status**: v0 — minimal aggregate-only schema. Per-user pseudonyms intentionally NOT stored.

## v0 design principles (Codex-derived)

1. **Define dashboard metrics first, then retain only fields needed for those metrics.** Do not architect for "vague future dashboards."
2. **No per-user pseudonyms in v0.** Aggregate only. Per-user retention requires a privacy review and a concrete intervention design that needs A/B-style attribution.
3. **Collector is sensitive infrastructure.** Treat it as such: admin-only access, no debug logs in prod, no failed-payload retention >24h, no full-fidelity staging mirror, automated test that proves dropped fields stay dropped.

## v0 dashboard metrics (the only thing the collector needs to support)

These are the 6 metrics that drive recommendations updates and tell the story of org-wide burn:

1. **Model mix** — daily share of input + output tokens by model
2. **Token totals** — daily input, output, cache_read, cache_create totals (org-wide sums only)
3. **Cache ratio** — daily cache_read / (input + cache_read) ratio
4. **MCP load/invoke ratio** — per MCP server, ratio of "loaded into context" events vs. "actually invoked" events. Identifies idle MCPs org-wide.
5. **Tool-call counts** — daily count of tool invocations bucketed by tool_name
6. **Session length distribution** — daily histogram of session message count, bucketed: <10, 10-30, 30-80, 80+

That's it. Anything beyond this needs a written justification and a privacy review.

## What we DROP at ingest

| Field | Action | Why |
|---|---|---|
| `user.email` | DROP entirely (no hashing in v0) | We're not retaining per-user data |
| `user.id` | DROP | Same |
| `prompt.text`, `prompt.content`, `*.text` | DROP | Free-text content out of scope |
| `prompt.id`, `conversation.id` | DROP | Joinable to platform logs to re-ID |
| `file.path`, `file.contents` | DROP | Path may contain user dir name |
| `tool.arguments.*` (verbatim args) | DROP | Keep tool name; drop args |
| Exact `start_time` | TRUNCATE to day-bucket | Weakens side-channel re-ID |

## What we KEEP

| Field | Aggregation |
|---|---|
| `model` | grouped sum |
| `tool_name` | grouped count |
| `mcp_server.name` | grouped count of load + invoke events separately |
| `input_tokens`, `output_tokens` | sum |
| `cache_read_tokens`, `cache_creation_tokens` | sum |
| `duration_ms` | distribution (p50, p95) |
| `bucket_day` (truncated date) | grouping key |
| `session_msg_count_bucket` (<10/10-30/30-80/80+) | distribution |

NO `user_pseudonym` column in v0 schema. Add only if v1 has a documented intervention requiring it.

## otelcol config sketch

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
        # Cloudflare Access in front; bearer token verified at edge

processors:
  attributes/drop_pii:
    actions:
      - { key: user.email,       action: delete }
      - { key: user.id,          action: delete }
      - { key: prompt.text,      action: delete }
      - { key: prompt.content,   action: delete }
      - { key: prompt.id,        action: delete }
      - { key: conversation.id,  action: delete }
      - { key: file.path,        action: delete }
      - { key: file.contents,    action: delete }
      # tool argument scrub: keep tool.name, drop everything else under tool.*

  transform/day_bucket:
    trace_statements:
      - context: span
        statements:
          - set(attributes["bucket.day"], TruncTime(start_time, "1d"))

  filter/required_fields:
    # Drop spans that somehow still carry email or prompt.text after the drop_pii pass
    traces:
      span:
        - 'attributes["user.email"] != nil'
        - 'attributes["prompt.text"] != nil'

  batch:
    timeout: 30s

exporters:
  postgresql:
    endpoint: postgres://collector:5432/pago_otel
    sslmode: require
    # Schema: org_aggregate_daily(bucket_day, model, tool_name, mcp_server,
    #         load_events, invoke_events, input_tokens, output_tokens,
    #         cache_read_tokens, cache_creation_tokens, duration_p50_ms,
    #         duration_p95_ms, session_msg_count_bucket, count)

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors:
        - attributes/drop_pii
        - transform/day_bucket
        - filter/required_fields
        - batch
      exporters: [postgresql]
    metrics:
      receivers: [otlp]
      processors: [attributes/drop_pii, transform/day_bucket, filter/required_fields, batch]
      exporters: [postgresql]
```

(OTTL syntax is illustrative — verify against current `otelcol-contrib`. The `filter/required_fields` step is a belt-and-braces test that PII never lands.)

## Operational hardening (non-negotiable)

| Control | Setting |
|---|---|
| Access | Admin-only SSH via 1Password-gated cert. No human reads prod DB directly. |
| Log level (prod) | `warn` only. No raw payload logging anywhere. |
| Failed-payload retention | 24h max. After 24h, deleted. No "for debugging" exception. |
| Staging mirror | Synthetic test data only. Never mirror prod traffic at full fidelity. |
| Secret rotation | OTel ingress bearer token + DB password rotated quarterly. |
| Drop-test in CI | On every otelcol config change, replay a fixture that includes `user.email` + `prompt.text`; confirm zero rows in DB contain those fields. Fail the deploy if not. |
| Retention | 90 days for daily aggregates. Older data: keep monthly rollups only. |

## Anthropic Settings → Monitoring values

Once collector is up and the Cloudflare Access bearer token is set:

- **OTLP endpoint**: `https://collector.pago.ro/v1/traces`
- **OTLP protocol**: `http/protobuf`
- **OTLP headers**: `Authorization=Bearer <INGRESS_BEARER_TOKEN>`

Save. Verify with one test session: `psql` → confirm a row in `org_aggregate_daily` populated, no `user.email` anywhere.

## Verification checklist before going live

- [ ] `psql` confirms no row contains a string matching `*@pago.ro`
- [ ] `psql` confirms no row has any `prompt.text` or `prompt.id` field
- [ ] No `user_pseudonym` column exists in v0 schema (if it does, you regressed to v2)
- [ ] CI drop-test passes
- [ ] Cloudflare Access bearer token rotation procedure documented
- [ ] Quarterly rotation calendar event scheduled
- [ ] Opt-out mechanism documented for users (admin can exempt by `bucket_day + org_id` cohort, but since we don't track per-user, opt-out is "we drop your events on the wire if you ask" — implementation: per-source-IP filter, not per-pseudonym)

## v0 → v1 promotion criteria

Add per-user data ONLY when ALL of:
1. A specific intervention design is written that requires per-user attribution (e.g., "did the skill recommendation actually change R3 adoption?")
2. A privacy review is completed and signed off
3. The HMAC pipeline + opt-out per-pseudonym is added
4. The dashboard explicitly k-anonymizes (k≥5) at query time
5. A retention policy specifies max 90 days for any per-pseudonym row

Until all 5 land, v0 stays. Aggregates only.

## Dashboard queries (alex-only, v0)

```sql
-- Org-wide model mix, last 14 days
SELECT model, SUM(input_tokens) AS in_tok, SUM(output_tokens) AS out_tok
FROM org_aggregate_daily
WHERE bucket_day >= NOW() - INTERVAL '14 days'
GROUP BY model
ORDER BY in_tok DESC;

-- Org-wide cache ratio trend
SELECT bucket_day,
       SUM(cache_read_tokens)::float / NULLIF(SUM(input_tokens) + SUM(cache_read_tokens), 0) AS cache_ratio
FROM org_aggregate_daily
WHERE bucket_day >= NOW() - INTERVAL '14 days'
GROUP BY bucket_day
ORDER BY bucket_day;

-- Idle MCP servers (loaded but rarely invoked)
SELECT mcp_server,
       SUM(load_events) AS loaded,
       SUM(invoke_events) AS invoked,
       SUM(invoke_events)::float / NULLIF(SUM(load_events), 0) AS use_ratio
FROM org_aggregate_daily
WHERE bucket_day >= NOW() - INTERVAL '7 days'
GROUP BY mcp_server
HAVING SUM(load_events) >= 5
ORDER BY use_ratio ASC;

-- Session length distribution
SELECT session_msg_count_bucket, SUM(count) AS sessions
FROM org_aggregate_daily
WHERE bucket_day >= NOW() - INTERVAL '7 days'
GROUP BY session_msg_count_bucket
ORDER BY session_msg_count_bucket;
```

There are no per-user queries in v0. By design.
