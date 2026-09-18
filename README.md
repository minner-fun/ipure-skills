# IPure Skills — IP purity check for AI agents

Agent Skills that teach Claude Code, Codex CLI and other skill-aware agents to answer one question well:

> **Is this IP clean enough for what I'm about to do?**

They call the free [IPure](https://ipure.dev) API — no key, no sign-up — and turn the result into an answer a person can act on: a 0-100 purity score, residential vs datacenter, native vs broadcast, proxy / VPN / Tor detection (with the VPN operator when known), blocklists, abuse and attack history, and per-scenario suitability for **AI services, social sign-up, streaming, gaming platforms, cross-border e-commerce and email**. Every deducted point is traceable to a signal and a data source, and the skill always states what an IP check *cannot* tell you.

[中文说明](#中文说明)

## Skills

| Skill | Language | Use it for |
| --- | --- | --- |
| [`ip-purity-check`](skills/ip-purity-check/SKILL.md) | English | "Is this proxy clean?", "Can I use this IP for ChatGPT / X sign-up / Netflix / Steam?", "Which of these IPs is best?", pre-flight check of an agent's egress IP |
| [`ip-purity-check-zh`](skills/ip-purity-check-zh/SKILL.md) | 中文 | 「这个 IP 干不干净」「这个节点能不能登 ChatGPT」「是不是原生 IP / 住宅 IP」「这几个 IP 用哪个」 |

## Install

**Claude Code — as a plugin**

```
/plugin marketplace add minner-fun/ipure-skills
/plugin install ipure@ipure-skills
```

**Claude Code — copy the skill**

```bash
git clone https://github.com/minner-fun/ipure-skills
cp -r ipure-skills/skills/ip-purity-check ~/.claude/skills/        # personal
# or into a project: cp -r ipure-skills/skills/ip-purity-check .claude/skills/
```

**Codex CLI**

```bash
git clone https://github.com/minner-fun/ipure-skills
cp -r ipure-skills/skills/ip-purity-check ~/.codex/skills/
```

Any agent that reads `SKILL.md` files works the same way: drop the folder where your agent looks for skills. The only requirement is `curl` (and ideally `jq`).

## Try it

```
> I bought a proxy, exit IP is 146.70.132.85. Can I use it for ChatGPT?
> Check my current IP before we start scraping.
> Which of these is best for registering X accounts: 1.2.3.4, 5.6.7.8, 9.10.11.12
```

Or call the API yourself:

```bash
curl -s "https://ipure.dev/api/lookup?ip=8.8.8.8" | jq '{purity: .risk.purity, verdict: .risk.verdict, scenarios: [.scenarios[] | {id, score, levelLabel}], report: .reportUrl}'
```

## What the API gives you

- `risk.purity` (0-100, higher is cleaner), level, verdict, **and a separate `confidence`**
- `risk.factors[]` — where every point came from
- `usageType`, `nativeType`, `flags` (proxy / VPN / Tor / hosting / relay), `vpnOperator`
- `scenarios[]` — `ai`, `social`, `streaming`, `gaming`, `ecommerce`, `email`, each weighted differently
- `blocklists[]`, `abuse[]` (reports and attack history), `exposure`, `sharing`
- `sources[]` — which data sources answered and which failed (a failed source is not a clean result)
- `unknowns[]` — what cannot be determined from an IP alone
- `reportUrl` — the human-readable report to cite

Docs: <https://ipure.dev/docs/api> · OpenAPI: <https://ipure.dev/openapi.json> · For models: <https://ipure.dev/llms.txt> · Aggregate statistics: <https://ipure.dev/insights>

## Limits

Free, no API key. 30 requests per minute per client. IPs already in IPure's database return instantly; an IP that is not yet in the database triggers a live lookup, and unverified clients get 5 of those per source IP per day. When that runs out the API returns `403 verification_required` with a `reportUrl` — open it in a browser once and the IP is in the database from then on. Looking up your own egress IP (`/api/lookup` with no `ip`) never needs verification. Please do not bulk-scan ranges.

## Honest boundaries

An IP check describes IP-level risk from public and partner data at query time. It cannot see a platform's internal reputation data, whether an IP is truly exclusive, which accounts used it before, or your device fingerprint and behaviour. The skills are written to say so every time, and never to promise that an IP is "safe" or "won't be flagged".

## 中文说明

这是给 AI Agent 用的「IP 纯净度检测」技能，支持 Claude Code、Codex CLI 等能读取 `SKILL.md` 的 Agent。它调用免费的 [IPure](https://ipure.dev) 接口（无需 key、无需注册），回答一个问题：**这个 IP 拿去做我要做的事，够不够干净？**

返回 0-100 纯净度评分、住宅 / 机房识别、原生 / 广播判定、代理 / VPN / Tor 判定（已知商业 VPN 给出运营商）、黑名单、滥用与攻击记录，以及 AI 服务、社交平台注册、流媒体、游戏平台、跨境电商、邮件发送六个场景各自的适用性。每一分的扣分依据和数据来源都可追溯，并且每次都会说明「仅凭 IP 无法判断的事」。

安装（Claude Code）：

```
/plugin marketplace add minner-fun/ipure-skills
/plugin install ipure@ipure-skills
```

或直接复制技能目录：

```bash
git clone https://github.com/minner-fun/ipure-skills
cp -r ipure-skills/skills/ip-purity-check-zh ~/.claude/skills/
```

然后直接问：「我买了个代理，出口是 146.70.132.85，能登 ChatGPT 吗？」「开始爬之前先查一下我现在的出口 IP」。

接口文档：<https://ipure.dev/docs/api> · 数据洞察：<https://ipure.dev/insights>

## License

MIT
