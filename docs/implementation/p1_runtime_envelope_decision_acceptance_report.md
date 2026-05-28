# P1 架构归位验收报告

日期：2026-05-28  
项目：ai_theme_app → AlphaPilot / alpha_pilot

## P1-1 Runtime Lite

```
$ python -m runtime.cli status --profile realtime
🟢 redis (必)
🟢 postgres (必)
🟢 web_app_service (必)
🟢 sps (必)
🟢 jyhf_cdp_service
🟢 raw_news_services
🟢 phase0_decision_services
🟢 frontend_dev
Running: 8/8
```

## P1-2 Envelope + Stream + Stable run_id

| 模块 | 文件 | 验证 |
|------|------|------|
| Envelope Adapter | `core/contracts/envelope.py` | `ensure_envelope()` 旧消息包装，透传已有 envelope |
| Stream Alias | `core/contracts/streams.py` | `resolve("stream:intel.raw.news")` → `"stream:news:raw"` |
| run_id 稳定 | envelope.py | 同进程多次调用 `ensure_envelope()` run_id 不变 |

## P1-3 Decision 收口

| 阶段 | 内容 | 验证 |
|------|------|------|
| P1-3.0 | SignalDecision 结构定义 | `core/contracts/decision.py` — level / scores / evidence / risk_flags |
| P1-3.1 | SSE 端点试点接入 | Kline `_decision.decision_type=support_alert`，W2S `_decision.decision_type=w2s_alert`。Legacy 字段保留，facade 失败不影响数据流 |

## 下一步

P1-3.2：Decision 消费侧试点 — Intel Feed / debug endpoint 读取 `_decision`，legacy fallback。
