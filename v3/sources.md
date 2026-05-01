# Pago Optimizer — Sources & Citation Tiers

Every claim in the skill maps to a source here. Tiers gate how confidently we present a recommendation.

## Tiers

| Tier | Definition | Allowed in skill output? |
|---|---|---|
| `[vendor]` | Anthropic official docs, pricing page, support articles | Yes |
| `[community-replicated]` | Same finding from ≥3 independent reputable sources | Yes |
| `[community-single]` | Single article / blog post, not yet replicated | NO — footnote only |
| `[independent]` | Peer-reviewed research, independent measurement | Yes |
| `[telemetry]` | Pago's own collector data with k≥5 cohort | Yes |

## Active sources (used in recommendations.md)

### Anthropic vendor docs
- Pricing: https://platform.claude.com/docs/en/about-claude/pricing
- Best practices: https://code.claude.com/docs/en/best-practices
- Cost management: https://code.claude.com/docs/en/costs
- Extended thinking tips: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/extended-thinking-tips
- Prompt caching: https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- Skills reference: https://code.claude.com/docs/en/skills
- Hooks reference: https://code.claude.com/docs/en/hooks
- Cowork OTel monitoring: https://support.claude.com/en/articles/14477985-monitor-claude-cowork-activity-with-opentelemetry
- Skills org sharing: https://support.claude.com/en/articles/13119606-provision-and-manage-skills-for-your-organization
- Cowork getting started: https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork
- Usage & cost API: https://platform.claude.com/docs/en/api/usage-cost-api
- Claude Code analytics API: https://platform.claude.com/docs/en/build-with-claude/claude-code-analytics-api

## Demoted footnotes (NOT used in recommendations)

### Claude Code v2.1.x prefix-cache bugs
- Source: theregister.com 2026-03-31 — [community-single]
- Claim: "10–20× inflation if unpatched"
- Status: DEMOTED. Single source, not corroborated. Do NOT surface in skill output. Re-evaluate quarterly; promote if a second independent source confirms.

### Productivity uplift claims
- Various community blog posts and Anthropic marketing-adjacent docs
- Claim: "2–4× velocity uplift for power users"
- Status: NOT USED. Selection-effect risk, no control group, not falsifiable from a 9-question intake. Do NOT claim outcomes the skill cannot measure.

## Quarterly source audit checklist

- [ ] All vendor URLs resolve and content matches the cited claim
- [ ] Pricing claims still accurate (Sonnet < Opus per input token, etc.)
- [ ] Any `[community-single]` source promoted to `[community-replicated]`? If so, move to active list.
- [ ] Any active claim now contradicted? Falsify and update `recommendations.md`.
- [ ] Add new sources discovered in Pago telemetry (`[telemetry]`) for any pattern observed in ≥5 distinct user-pseudonym days.
