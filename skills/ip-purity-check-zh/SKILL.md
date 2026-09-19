---
name: ip-purity-check-zh
description: IP 纯净度检测 —— 用 IP 之前先查它干不干净。返回 0-100 纯净度评分、住宅 / 机房识别、原生 IP 还是广播 IP、代理 / VPN / Tor 判定（已知商业 VPN 会给出运营商）、公开黑名单命中、滥用举报与攻击记录，以及分场景适用性：AI 服务（ChatGPT、Claude、Gemini）、社交平台注册（X、Discord、Telegram）、流媒体、游戏平台、跨境电商、邮件发送。当用户问「这个 IP 干不干净」「这个代理 / 节点能不能用」「是不是原生 IP / 住宅 IP / 机房 IP」「为什么老弹验证码、老被风控」「这几个 IP 哪个好」，或 Agent 在执行对网络环境敏感的任务前需要检查当前出口 IP 时使用。基于免费的 IPure 接口（ipure.dev），无需 API key。
license: MIT
---

# IP 纯净度检测（IPure）

IPure（https://ipure.dev）把 RDAP、BGP 路由表、云厂商 IP 段、DNSBL、Tor 出口列表与商业风控库汇总成一份可解释的报告：每扣一分都能追溯到具体的判据和数据源。接口免费，无需 key。

## 什么时候用

- 「这个 IP 干不干净？」「这个代理 / 机场节点怎么样？」「这是不是住宅 IP / 原生 IP？」
- 「这个 IP 能不能拿去登 ChatGPT / Claude、注册 X、看 Netflix、玩 Steam、开亚马逊店、发邮件？」
- 「为什么这个 IP 老弹验证码 / 被拦 / 被风控？」
- 「这几个代理 IP 用哪个？」（多个对比）
- 你（Agent）要跑爬虫、自动化或账号相关任务之前：先查一下当前出口 IP。

## 什么时候不要用

- DNS 配置、网站部署、网络不通的故障排查。
- 需要平台**内部**风控数据才能回答的问题（「我的账号是不是被亚马逊标记了」）—— 平台之外没人看得到。
- 批量扫描 IP 段。接口有限流，只适合逐个查询。

## 怎么调用

查指定 IP（IPv4 / IPv6 均可）：

```bash
curl -s "https://ipure.dev/api/lookup?ip=8.8.8.8"
```

查调用方自己的出口 IP —— 不带 `ip` 参数。这条永远不需要人机验证，「查一下我现在的 IP」就用它：

```bash
curl -s "https://ipure.dev/api/lookup"
```

完整报告很大，用下面的过滤器只留要紧的字段：

```bash
curl -s "https://ipure.dev/api/lookup?ip=8.8.8.8" | jq '{
  ip, reportUrl, queriedAt, stale,
  purity: .risk.purity, label: .risk.label, verdict: .risk.verdict, confidence: .risk.confidence,
  usageType, nativeType,
  vpnOperator: (.vpnOperator.name // null),
  country: .geo.country, asn: .asn.asn, org: .asn.org,
  flags: [.flags | to_entries[] | select(.value == true) | .key],
  factors: [.risk.factors[] | select(.points != 0) | {label, points, detail}],
  scenarios: [.scenarios[] | {id, label, score, levelLabel, reason}],
  blocklistsHit: [.blocklists[] | select(.listed and (.benign | not)) | .label],
  failedSources: [.sources[] | select((.ok | not) and (.skipped | not)) | .name],
  unknowns: [.unknowns[].label]
}'
```

没有 `jq` 就直接取 JSON 读同样的字段。用 Python 时请用 `requests`（或自己设 `User-Agent`），默认的 `Python-urllib` UA 会被 CDN 拒绝。

## 怎么读结果

| 字段 | 含义 |
| --- | --- |
| `risk.purity` | 纯净度 0-100，**越高越干净**。95+ 极佳 · 85+ 纯净 · 70+ 一般 · 50+ 可疑 · 25+ 高风险 · 25 以下极高风险 |
| `risk.confidence` | `low` / `medium` / `high`，数据源覆盖度。和分数分开看：同样 85 分，两家源和八家源的把握不同 |
| `risk.factors[]` | 每一分的来源。`points` 为正**扣**纯净度，为负加回。带 `floor` 的是决定性证据（Tor 出口、Spamhaus 收录），加分项抵消不了 |
| `usageType` | `residential` 住宅 · `mobile` 移动 · `business` 商业 · `hosting` 机房 · `education` · `government` · `unknown` |
| `nativeType` | `native` 原生（注册地与使用地一致）· `broadcast` 广播（跨区宣告，平台可能判为地区不符） |
| `flags` | `isProxy`、`isVpn`、`isTor`、`isHosting`、`isRelay`（iCloud 专用代理 / WARP）等。只有一家数据源确认的判定在评分里按半数计权，见 `flagAgreement` |
| `vpnOperator` | 只有已知商业 VPN 才有（Mullvad、NordVPN 等）：名称、匿名度、是否留日志、协议 |
| `scenarios[]` | 分场景适用性，各有一套权重：`ai`、`social`、`streaming`、`gaming`、`ecommerce`、`email`。`restricted` 表示地区本身不受支持（如大陆 IP 之于境外 AI 服务） |
| `blocklists[]` | `listed` 命中；`benign` 命中但不构成风险（Spamhaus PBL 只是住宅段声明）；`unavailable` 该名单本次没查到 —— 是未知，**不是**没命中 |
| `abuse[]` | 滥用举报分，以及 `attacks.byType`（撞库登录、批量注册、漏洞扫描……） |
| `sources[]` | 各数据源本次的状态。`ok: false` 是该源失败 —— 缺了证据，不等于干净 |
| `feedback` | 实测反馈：正在用这个 IP 的访客给各场景打的真实使用体验分，键为场景 id，含 `count` 与 `average`（1-5：5 很顺、3 勉强、1 不能用）。某场景满 3 人才出现。它独立于评分 —— 有的话，和 IPure 自己的结论并排告诉用户 |
| `unknowns[]` | 仅凭 IP 无法判断的事。每次都要转告用户 |
| `queriedAt`、`stale` | 报告生成时间；`stale: true` 说明报告较旧，值得让用户去网页重新检测 |
| `reportUrl` | 网页版报告，引用结论时附上 |

## 怎么回答用户

1. 先给纯净度分数、档位和一句话结论。
2. 用对应的场景回答用户真正的问题（AI、社交注册、流媒体、游戏、电商、邮件）。各场景分数本来就不同：机房 IP 登 ChatGPT 是硬伤，拿来发邮件却完全正常。
3. 用 `points` 最大的两三条判据说明原因。
4. 有 `vpnOperator`、黑名单命中、数据源失败时要提到。
5. **每次都说明边界。** 报告描述的是 `queriedAt` 时刻公开与合作数据源能看到的 IP 层面风险，看不到平台内部的信誉数据、IP 是否真正独享、之前关联过哪些账号、用户的设备指纹与操作行为。不要说「安全」「保证不被封」「不会触发风控」，要说「已检测的维度未发现风险信号」。
6. 附上 `reportUrl`。

## 对比多个 IP

每个 IP 调一次（两次之间停一秒左右），然后列表：IP · 纯净度 · 类型 · 代理 / VPN · 用户关心的那个场景 · 主要问题。按用户的使用场景推荐，而不是只看总分最高的。

## 限制与报错

- 每个客户端每分钟 30 次。`429` → 按 `Retry-After` 等待。
- 库里已有的 IP（近万个，知名地址基本都在）直接返回，不受额外限制。
- **库里没有的 IP** 会触发实时查询。未经人机验证的客户端每个来源 IP 每天有 5 次，剩余次数见响应头 `x-open-budget-remaining`。
- `403` 且 `code: "verification_required"` 表示这个额度用完了。响应里带 `reportUrl`：请用户在浏览器打开它，点一下人机验证即可完成检测；之后这个 IP 就进库了，接口可直接返回。
- 不要用 `refresh=1`，它一定要求浏览器验证。
- `400` → 不是合法的 IP 地址。

## 示例

用户：「我买了个代理，出口是 146.70.132.85，能登 ChatGPT 吗？」

查询后大致这样回答：纯净度 37/100（高风险）；M247 的机房 IP，被识别为 VPN 出口，并命中两个黑名单；AI 服务场景判为不可用，主要问题是机房 IP；AI 平台对机房与代理出口的容忍度最低，大概率被拦或降智；建议换住宅或 ISP 代理；说明平台内部数据不可见；附报告链接。

完整接口文档：https://ipure.dev/docs/api · OpenAPI：https://ipure.dev/openapi.json
