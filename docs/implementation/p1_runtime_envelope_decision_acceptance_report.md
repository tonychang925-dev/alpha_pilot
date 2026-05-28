# P1 架构归位验收报告（最终版）

日期：2026-05-28  
项目：ai_theme_app → AlphaPilot / alpha_pilot

## 交付总览

| 阶段 | 内容 | 文件 | 状态 |
|------|------|------|------|
| P1-1 | Runtime Lite status/health CLI | `runtime/cli.py`, `health.py`, `profiles/realtime.yaml` | ✅ merged |
| P1-2 | Envelope Adapter + Stream Alias | `core/contracts/envelope.py`, `streams.py` | ✅ merged |
| P1-2.1 | run_id per-process stable | `core/contracts/envelope.py` | ✅ merged |
| P1-3.0 | SignalDecision facade | `core/contracts/decision.py` | ✅ merged |
| P1-3.1 | SSE producer pilot (Kline/W2S) | `api_app.py` (SPS) | ✅ merged |
| P1-3.2 | Consumer endpoint `/api/v1/decision/latest` | `api_app.py` (SPS) | ✅ merged |
| P1-3.3 | Frontend debug panel | `RealtimeCollectorPage.tsx`, `api.ts`, `routes.py` | ✅ PR open |
| P1-3.4 | Intel Feed decision wrapping | `routes.py` (BFF) | ✅ PR open |
| P1-3.5 | Auto scores extraction | `core/contracts/decision.py` | ✅ PR open |

## 核心模块

### core/contracts/ — 统一契约层

| 模块 | 行数 | 功能 |
|------|------|------|
| `envelope.py` | 130 | `ensure_envelope()` 旧消息包装 + `get_payload()` 提取 + 稳定 run_id |
| `streams.py` | 32 | `resolve()` 新→旧 alias 映射 |
| `decision.py` | 220 | `SignalDecision` dataclass + `ensure_decision()` facade + auto scores |

### runtime/ — 运行控制层

| 模块 | 行数 | 功能 |
|------|------|------|
| `cli.py` | 92 | `status` / `health` 命令 |
| `health.py` | 92 | TCP / HTTP / process 三种探针 |
| `profiles/realtime.yaml` | 68 | 8 个服务声明 |

### Decision 全链路

```
Producer (SSE) → Redis Stream → Consumer API → Frontend Panel
     ↓                              ↓
  _decision 字段               /api/v1/decision/latest
  + legacy 保留                + frontend debug card
     ↓                              ↓
  Kline/W2S/Intel Feed         Intel Feed _decision
```

### Scores 自动提取

| 告警类型 | 提取字段 |
|----------|---------|
| support_alert | support_strength, distance_pct, confidence |
| w2s_alert | confirm_score, d2_score, relative_strength |
| event | impact_score, confidence |
| 所有 | final_score（加权平均） |

## 验收记录

```
$ python -m runtime.cli status    →  8/8 🟢
$ curl /api/v1/decision/latest    →  SignalDecision[] with scores
$ curl /api/v2/intel/stream       →  _decision in every item
```

## 下一步

P2：领域引擎 Facade（MarketStateEngine / SupportEngine / W2SEngine / DecisionEngine）
