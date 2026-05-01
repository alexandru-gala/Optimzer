# Pago OTel Collector — Setup Notes

**Audience**: Pago admin (alex) configuring the OTLP endpoint that the Anthropic Settings → Monitoring panel will point at for Office/Cowork agents.
**Status**: Draft — verify event field names against current Anthropic Cowork OTel docs before deploying.

## What this collector exists to do

Anthropic's Team plan exposes an OTLP push endpoint setting under **Settings → Monitoring**:

- OTLP endpoint (e.g. `https://collector.pago.ro`)
- OTLP protocol (`http/protobuf` or `grpc`)
- OTLP headers (`Authorization=Bearer <token>`)

When configured, Office/Cowork agents push OpenTelemetry traces and metrics for every session: prompts, tool/MCP invocations, file access, approvals, token counts, costs, durations. The collector receives the raw stream, **anonymizes at ingest**, and writes only safe aggregates to storage. The dashboard reads from storage — never from the raw ingress.

## Privacy framing (read first)

This is **pseudonymization**, not anonymization. The HMAC + drop pipeline reduces re-identification risk but at N=50 users it does not eliminate it. We commit to:

1. Never query at per-pseudonym granularity outside cohorts of k≥5.
2. Rotate the HMAC secret quarterly (re-pseudonymizes everyone — historical pseudonyms become unjoinable to current ones).
3. Document the residual re-ID risk in the team announcement.
4. Make opt-out trivial: a per-user exemption list at the collector drops events for any user who asks.

## Architecture

```
Anthropic Office/Cowork client
        │
        │ OTLP/HTTPS (config: Settings → Monitoring)
        ▼
Cloudflare Access (ingress auth: bearer token check)
        │
        ▼
otelcol-contrib (custom processor pipeline)
        │
        ├─ filter/drop-pii processor: drop user.email, prompt.text, prompt.id
        ├─ transform/hmac processor:   user.email → user_pseudonym (HMAC-SHA256)
        ├─ filter/opt-out processor:   drop events for opted-out pseudonyms
        ├─ groupbyattrs processor:     bucket by day + user_pseudonym + model
        └─ exporter:                   Postgres (small) or ClickHouse (if scale grows)
```

## otelcol config (sketch)

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Drop free-text fields entirely
  attributes/drop_pii:
    actions:
      - key: prompt.text
        action: delete
      - key: prompt.id
        action: delete
      - key: file.contents
        action: delete

  # Hash user.email → user_pseudonym, then drop the original
  transform/hmac_email:
    trace_statements:
      - context: span
        statements:
          - set(attributes["user_pseudonym"], SHA256(Concat([attributes["user.email"], "${env:HMAC_SECRET}"], "")))
          - delete_key(attributes, "user.email")

  # Drop events for opted-out users
  filter/opt_out:
    traces:
      span:
        - 'attributes["user_pseudonym"] == "<opted-out-hash-1>"'
        - 'attributes["user_pseudonym"] == "<opted-out-hash-2>"'

  # Truncate timestamps to day-bucket to weaken side-channel re-ID
  transform/day_bucket:
    trace_statements:
      - context: span
        statements:
          - set(attributes["bucket.day"], TruncTime(start_time, "1d"))

  batch:
    timeout: 30s

exporters:
  postgresql:
    endpoint: postgres://collector:5432/pago_otel
    sslmode: require

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors:
        - attributes/drop_pii
        - transform/hmac_email
        - filter/opt_out
        - transform/day_bucket
        - batch
      exporters: [postgresql]
    metrics:
      receivers: [otlp]
      processors: [attributes/drop_pii, transform/hmac_email, filter/opt_out, batch]
      exporters: [postgresql]
```

(The OTTL transform syntax above is illustrative — verify against the current `otelcol-contrib` version. SHA256 + Concat are real OTTL functions; HMAC-as-such may need a custom processor instead, in which case run a tiny Go shim.)

## Required env

- `HMAC_SECRET` — at least 32 random bytes, stored in 1Password, rotated quarterly. Rotation re-pseudonymizes everyone (intentional).
- `INGRESS_BEARER_TOKEN` — the value Anthropic posts in the OTLP `Authorization` header. Set in Cloudflare Access.
- `POSTGRES_DSN` — collector's writer DSN.

## Hosting options (cheapest first)

1. **Fly.io machine + Fly Postgres** — ~$10/mo for 50 users at expected event volume. Cloudflare Access in front. Simplest.
2. **Self-hosted on existing Pago infra** — if there's already a Postgres + ingress story, drop it there.
3. **Honeycomb / Grafana Cloud** — managed sink, but anonymization processor still needs to run before they see the data, so you'd run otelcol locally regardless. Save for "scale moment" not for now.

## Anthropic Settings → Monitoring values to paste

After collector is up:

- **OTLP endpoint**: `https://collector.pago.ro/v1/traces` (and `/v1/metrics` if separately configurable)
- **OTLP protocol**: `http/protobuf` (simpler than gRPC behind Cloudflare Access)
- **OTLP headers**: `Authorization=Bearer <INGRESS_BEARER_TOKEN>`

Click Save. Run a single test session in Cowork. Verify in `pago_otel.spans` that a row appeared with `user_pseudonym` set and `user.email` absent.

## Verification checklist before going live

- [ ] `psql` confirms no row contains a string matching `*@pago.ro`
- [ ] `psql` confirms no row has a `prompt.text` or `prompt.id` field
- [ ] All `user_pseudonym` values are 64-char hex (SHA256 output)
- [ ] Opt-out filter drops a known test user's events
- [ ] HMAC secret rotation procedure documented
- [ ] Quarterly rotation calendar event scheduled

## What NOT to log to storage

These fields are present on the wire but MUST be stripped at ingest:

- `user.email` (strip after hashing)
- `prompt.text`, `prompt.content`, any `*.text` field on user-generated content
- `prompt.id`, `conversation.id` (joinable to the platform's own logs to re-identify)
- `file.path` if it contains a user's personal directory name
- Tool call arguments verbatim — keep tool name, drop args

## Dashboard queries (alex-only)

Run only k≥5 cohort queries. Examples:

```sql
-- Top model mix across the org, last 14d
SELECT model, COUNT(*) AS sessions, SUM(input_tokens) AS in_tok, SUM(output_tokens) AS out_tok
FROM spans
WHERE bucket_day >= NOW() - INTERVAL '14 days'
GROUP BY model
HAVING COUNT(DISTINCT user_pseudonym) >= 5
ORDER BY in_tok DESC;

-- Cache hit rate distribution
SELECT
  CASE
    WHEN cache_read / NULLIF(input_tokens, 0) > 0.5 THEN 'high (>50%)'
    WHEN cache_read / NULLIF(input_tokens, 0) > 0.2 THEN 'medium'
    ELSE 'low (<20%)'
  END AS cache_bucket,
  COUNT(DISTINCT user_pseudonym) AS users
FROM spans
WHERE bucket_day >= NOW() - INTERVAL '7 days'
GROUP BY cache_bucket
HAVING COUNT(DISTINCT user_pseudonym) >= 5;

-- MCP servers loaded but never invoked
-- (load events vs invoke events per server, by user_pseudonym, aggregated)
```

Per-user drilldown queries are deliberately not provided. If you ever need to debug a specific user's quota, ask the user — don't query the collector.
