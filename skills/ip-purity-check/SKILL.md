---
name: ip-purity-check
description: Check how "clean" an IP address is before using it — purity score (0-100), residential vs datacenter, native vs broadcast, proxy / VPN / Tor detection (with the VPN operator when known), public blocklists, abuse and attack history, and per-scenario suitability for AI services (ChatGPT, Claude, Gemini), social platform sign-up (X, Discord, Telegram), streaming, gaming platforms, cross-border e-commerce and email sending. Use when the user asks whether an IP or proxy is clean, safe to use, residential, a VPN or datacenter IP, why an IP keeps triggering captchas or risk control, which of several proxy IPs is best, or before an agent runs a network-sensitive task through its current egress IP. Powered by the free IPure API (ipure.dev), no API key required.
license: MIT
---

# IP purity check (IPure)

IPure (https://ipure.dev) aggregates RDAP, BGP, cloud provider ranges, DNSBLs, the Tor exit list and commercial risk databases into one explainable report: every point deducted is traceable to a specific signal and data source. The API is free and needs no key.

## When to use this skill

- "Is this IP clean?" / "Is this proxy any good?" / "Is this a residential IP?"
- "Can I use this IP for ChatGPT / Claude / X sign-up / Netflix / Steam / an Amazon store / sending email?"
- "Why does this IP keep getting captchas / blocked / flagged?"
- "Which of these proxy IPs should I use?" (compare several)
- Before you (the agent) run a scraping, automation or account-related task: check the current egress IP first.

## When NOT to use it

- DNS configuration, website deployment, connectivity troubleshooting (ping / traceroute problems).
- Questions that need a platform's *internal* risk data ("is my account flagged by Amazon?") — nobody outside the platform can see that.
- Bulk scanning of IP ranges. The API is rate-limited and meant for individual lookups.

## How to call

Look up a specific IP (IPv4 or IPv6):

```bash
curl -s "https://ipure.dev/api/lookup?ip=8.8.8.8"
```

Look up the caller's own egress IP — omit `ip`. This never requires verification, so it is the right call for "check my current IP":

```bash
curl -s "https://ipure.dev/api/lookup"
```

The full report is large. Pipe it through this filter to keep only what matters:

```bash
curl -s "https://ipure.dev/api/lookup?ip=8.8.8.8" | jq '{
  ip, reportUrl, queriedAt, stale,
  purity: .risk.purity, label: .risk.label, verdict: .risk.verdict, confidence: .risk.confidence,
  usageType, nativeType,
  vpnOperator: (.vpnOperator.name // null),
  country: .geo.country, asn: .asn.asn, org: .asn.org,
  flags: [.flags | to_entries[] | select(.value == true) | .key],
  factors: [.risk.factors[] | select(.points != 0) | {label, points, detail}],
  scenarios: [.scenarios[] | {id, score, levelLabel, reason}],
  blocklistsHit: [.blocklists[] | select(.listed and (.benign | not)) | .label],
  failedSources: [.sources[] | select((.ok | not) and (.skipped | not)) | .name],
  unknowns: [.unknowns[].label]
}'
```

If `jq` is unavailable, fetch the JSON and read the same fields. With Python use `requests` (or set a `User-Agent`); the default `Python-urllib` user agent is rejected by the CDN.

## Reading the result

Text fields (`label`, `verdict`, `detail`, `reason`, `levelLabel`) are in Chinese — translate them into the user's language.

| Field | Meaning |
| --- | --- |
| `risk.purity` | 0-100, **higher is cleaner**. 95+ pristine · 85+ clean · 70+ neutral · 50+ suspicious · 25+ risky · below 25 dangerous (`risk.level`) |
| `risk.confidence` | `low` / `medium` / `high` — data-source coverage. Separate from the score: the same 85 means less with two sources than with eight |
| `risk.factors[]` | Where every point came from. `points` > 0 **lowers** purity, `points` < 0 raises it. `floor` marks decisive evidence (Tor exit, Spamhaus listing) that positive signals cannot offset |
| `usageType` | `residential` · `mobile` · `business` · `hosting` · `education` · `government` · `unknown` |
| `nativeType` | `native` (registered and used in the same country) · `broadcast` (announced cross-region — platforms may see a location mismatch) |
| `flags` | `isProxy`, `isVpn`, `isTor`, `isHosting`, `isRelay` (iCloud Private Relay / WARP), … A flag confirmed by a single source is scored at half weight — see `flagAgreement` |
| `vpnOperator` | Present only for known commercial VPNs (Mullvad, NordVPN, …): name, anonymity, logging policy, protocols |
| `scenarios[]` | Suitability per use case, each with its own weighting: `ai`, `social`, `streaming`, `gaming`, `ecommerce`, `email`. Levels: excellent · good · fair · poor · unusable · `restricted` (the region itself is not served, e.g. AI services from mainland China) · `not_applicable` (see next row). `score` is `null` for those two levels |
| `scenarioApplicable` | `false` means the address is public infrastructure (a public DNS resolver, a search-engine crawler, …), not anyone's egress IP, so per-scenario suitability does not apply; the reason is in `scenarioNote`. Report purity and ownership only — never say it is "suitable" for a use case |
| `blocklists[]` | `listed` = hit. `benign` = hit that is not a risk (Spamhaus PBL only declares a residential range). `unavailable` = the list could not be queried — unknown, **not** clean |
| `abuse[]` | Abuse-report score and `attacks.byType` (login attempts, registration attempts, vulnerability probing…) |
| `sources[]` | Per-source status. `ok: false` means that source failed — missing evidence, not a clean result |
| `feedback` | Real-world experience ratings from visitors who were actually using this IP, keyed by scenario id: `count` and `average` (1-5; 5 = works smoothly, 3 = usable with frequent captchas, 1 = does not work). A scenario appears only once it has 3+ ratings. Independent of the score — when present, quote it next to IPure's own verdict |
| `unknowns[]` | What cannot be determined from an IP alone. Always pass these on |
| `queriedAt`, `stale` | When the report was generated; `stale: true` means it is old enough that a re-check in the browser is worthwhile |
| `reportUrl` | The human-readable report. Cite it |

## How to present the answer

1. Lead with the purity score, its level and the one-line verdict.
2. Answer the user's actual question with the matching scenario (AI, social sign-up, streaming, gaming, e-commerce, email) — scenario scores differ on purpose: a datacenter IP is terrible for ChatGPT and perfectly normal for sending email.
3. Give the two or three factors with the largest `points` as the reason.
4. Mention `vpnOperator` when present, blocklist hits, and any failed sources.
5. **Always state the boundary.** The report describes IP-level risk from public and partner data at `queriedAt`. It cannot see a platform's internal reputation data, whether the IP is truly exclusive, which accounts used it before, or the user's device fingerprint and behaviour. Never say an IP is "safe", "guaranteed" or "will not be flagged" — say "no risk signals were found in the checked dimensions".
6. Link `reportUrl`.

## Comparing several IPs

Call the API once per IP (pause about a second between calls), then present a table: IP · purity · type · proxy/VPN · the scenario the user cares about · main problem. Recommend the best one for the user's scenario, not simply the highest overall score.

## Limits and errors

- 30 requests per minute per client. `429` → wait for `Retry-After`.
- IPs already in IPure's database (thousands, including most well-known addresses) return instantly and free of any limit beyond the rate limit.
- An IP **not yet in the database** triggers a live lookup. Unverified clients get 5 of those per source IP per day (remaining count in the `x-open-budget-remaining` response header).
- `403` with `code: "verification_required"` means that allowance is used up. The response includes `reportUrl`: ask the user to open it in a browser, which runs the check after a one-click human verification. After that the IP is in the database and the API returns it directly.
- Do not use `refresh=1`; it always requires browser verification.
- `400` → not a valid IP address.

## Example

User: "I bought a proxy, exit IP is 146.70.132.85 — can I use it for ChatGPT?"

Run the lookup, then answer along these lines: purity 37/100 (risky); datacenter IP at M247 identified as a VPN exit and listed on two blocklists; the AI-services scenario is rated unusable, main problem: datacenter IP; AI platforms are the least tolerant of datacenter and proxy exits, so expect blocks or degraded service; suggest a residential or ISP proxy instead; note that platform-internal data is not visible; link the report.

Full API reference: https://ipure.dev/docs/api · OpenAPI: https://ipure.dev/openapi.json
