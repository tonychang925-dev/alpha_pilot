# AlphaPilot 投研副驾 架构优化设计文档 v1.0

版本日期：2026-05-28
适用范围：实时采集、实时行情、实时事件、支撑告警、弱转强/W2S 告警、盘前必读、盘中情报台、盘后复盘

---

## 1. 文档目标

当前项目已经从“新闻事件 → 题材匹配”演化为“实时投资情报 + 盘前/盘中/盘后交易辅助”系统。功能增长很快，但架构边界、运行入口、数据契约、服务治理、告警分层没有同步收敛。

本文档目标：**在保留现有可用能力的基础上，完成一次低风险、可分阶段提交、每一步可验收的架构归位。**

核心原则：
- 不推倒重来
- 不改业务逻辑的 P0 修复先行
- 每一步独立可验收
- 迁移策略明确（Alias → 双写 → 切读 → 废弃）

---

## 2. 当前系统核心矛盾

1. **运行入口过多**：多个 shell 脚本 + Python manager + 手工命令，不知道当前到底启动了哪些服务
2. **BFF 单点故障**：port 8000 集 SPA 服务器 + API 路由 + CDP 进程管理 + SSE 代理 + 认证中间件于一身，一旦阻塞全部瘫痪
3. **进程僵尸泄漏**：竞价采集器每次启动创建新进程，旧的未清理；CDP 进程 BFF 重启后失去追踪
4. **资源失控**：raw_news_services 2.7GB 内存无上限；phase0_decision_services 59% CPU 无背压
5. **前端连接风暴**：React StrictMode 导致 4 个 SSE 连接（应为 2）；4 个独立轮询 = 375+ req/min 打向 BFF
6. **SSE 代理多余**：BFF 逐行转发 SPS 的 SSE 流，纯浪费一跳和一个长连接
7. **事件契约未统一**：不同模块使用不同字段名（source/source_type/source_channel/event_id/news_id/stock_code/symbol），无法全链路追踪
8. **告警边界不清**：支撑告警、W2S 告警、竞价观察散落在各脚本，缺统一分层和决策
9. **实时/回测链路未统一**：回测脚本和实时引擎不是同一套逻辑

---

## 3. 服务端口清单

| 服务 | 端口 | Python 环境 | 角色 |
|------|------|-------------|------|
| web_app_service（legacy BFF） | 8000 | .venv | SPA 服务 + API 路由 + CDP/auction 进程管理 + SSE 代理 + JWT 认证 |
| frontend_bff（intel BFF） | 8003 | .venv | Intel 专用 BFF |
| theme_service | 8002 | .venv | 题材服务 |
| stock_processing_service（SPS） | 8090 | theme_matcher_env | 数据处理 API + 实时管线控制 + SSE alert 源 + review queue |
| jyhf_cdp_service | 8095 | .venv | JYHF DOM CDP 采集（BFF 管理生命周期） |
| frontend dev server（Vite） | 5173 | node | 前端开发服务器 |
| Redis | 6379 | system | Stream 消息总线 |
| PostgreSQL | 5432 | system | stock_data_test 持久化 |

---

## 4. 优化后的总体架构（七层）

```text
┌──────────────────────────────────────────────┐
│  L7 UI / BFF 展示层                            │
│  Intel页面 / 实时采集页面 / 盘前必读 / 盘后复盘 │
└──────────────────────────────────────────────┘
                    ↑
┌──────────────────────────────────────────────┐
│  L6 Report & Feed 层                           │
│  IntelFeedBuilder / PremarketBriefing / Recap │
└──────────────────────────────────────────────┘
                    ↑
┌──────────────────────────────────────────────┐
│  L5 Decision Engine 决策层                     │
│  SignalDecisionEngine / AlertRouter / RiskGate │
└──────────────────────────────────────────────┘
                    ↑
┌──────────────────────────────────────────────┐
│  L4 Domain Engines 领域引擎层                  │
│  EventEngine / ThemeEngine / MarketState       │
│  SupportEngine / W2SEngine / StockPoolEngine   │
└──────────────────────────────────────────────┘
                    ↑
┌──────────────────────────────────────────────┐
│  L3 Stream Bus 实时数据总线                    │
│  Redis Streams + DLQ + Consumer Groups         │
└──────────────────────────────────────────────┘
                    ↑
┌──────────────────────────────────────────────┐
│  L2 Normalize / Quality / Dedup 标准化层       │
│  Envelope / Normalizer / Prefilter / Deduper    │
└──────────────────────────────────────────────┘
                    ↑
┌──────────────────────────────────────────────┐
│  L1 Source Adapters 数据源适配层               │
│  AkShare / CLS / JYHF DOM / TDX / JYHF Market  │
└──────────────────────────────────────────────┘
                    ↑
┌──────────────────────────────────────────────┐
│  L0 Runtime Control Plane 运行控制层           │
│  Orchestrator / Config / Health / Status / CLI │
└──────────────────────────────────────────────┘
```

---

## 5. 三层执行路线

### 第一层：P0 生产稳定性修复（止血，本周完成）

**原则：不改业务逻辑、不改 Stream 契约、不引入新框架、只解决当前系统稳定性问题。**

#### P0 不可越界原则

P0 阶段只允许修改稳定性问题，**不允许**做以下事情：

1. 不新增业务策略
2. 不重构 ThemeProcessor / DecisionExecutor
3. 不迁移 Stream 名称
4. 不新增 Runtime 框架
5. 不新增复杂数据库表
6. 不调整 W2S scorer 逻辑
7. 不改现有题材匹配主流程

**P0 的唯一目标是：系统不重复启动、不泄漏进程、不打爆 CPU/内存、不制造前端连接风暴。**

#### P0 执行顺序

严格按以下顺序执行，不做 Runtime，不做 Envelope，先把当前系统稳定下来：

```
P0-A1 → P0-A2 → P0-B1 → P0-B2 → P0-C1 → P0-C2 → P0-D1
```

#### P0-A：进程治理

| # | 任务 | 文件 | 说明 |
|---|------|------|------|
| P0-A1 | 竞价采集器加 parent_pid | `web_app_service/services/jyhf_auction_manager.py` | `_run_subprocess` 加 `--parent-pid`；start() 前 poll() 检查旧进程；stop() 前 ps 验证 PID |
| P0-A2 | CDP Manager 加 pidfile | `web_app_service/services/jyhf_cdp_manager.py` | start 时写 pidfile；`__init__` 时读 pidfile 恢复 ownership；BFF 重启不丢追踪 |

**验收**：
```bash
# 连续点击启动 3 次，只有 1 个竞价进程
ps aux | grep jyhf_auction | grep -v grep | wc -l  # = 1
# BFF 重启后 CDP 状态仍为 managed
curl -s http://127.0.0.1:8000/api/v2/realtime/jyhf-cdp/status | python -c "import sys,json; print(json.load(sys.stdin).get('service_owner'))"
```

#### P0-B：资源治理

| # | 任务 | 文件 | 说明 |
|---|------|------|------|
| P0-B1 | raw_news 内存保护 | `evaluate_service/e2e/pre_market_brief/run_raw_news_services.py` | 优先使用 psutil RSS 监控 + 超限优雅退出（`sys.exit(137)` 让 supervisor 重启）；`RLIMIT_AS` 作为可选硬限制（环境变量 `RAW_NEWS_ENABLE_HARD_LIMIT=1` 启用）。默认 `RAW_NEWS_MAX_MEMORY_MB=3072` |
| P0-B2 | phase0 CPU 限流 | `evaluate_service/e2e/pre_market_brief/run_phase0_decision_services.py` | 加 `--batch-delay-ms 100`，每批处理后 `await asyncio.sleep()` |

**验收**：
```bash
ps aux | grep raw_news | awk '{print $6}'  # RSS < 3GB
top -l 1 | grep phase0  # CPU < 40%
```

#### P0-C：连接治理

| # | 任务 | 文件 | 说明 |
|---|------|------|------|
| P0-C1 | 移除 React StrictMode | `frontend/src/main.tsx` | 删除 `<React.StrictMode>` 包裹，防止 useEffect 双重挂载 |
| P0-C2 | 合并前端轮询 | `web_app_service/api/routes.py` + `RealtimeCollectorPage.tsx` | 新增 `/api/v2/realtime/status-bundle`，并行获取 new-chain + CDP + auction 状态；前端 1 个 8s 轮询替代 3 个 |

**验收**：
```bash
# 浏览器 DevTools Network 面板：只有 2 个 SSE（非 4 个）
# 浏览器 DevTools Network 面板：只有 1 个状态轮询（非 3 个）
```

#### P0-D：SSE 链路瘦身

| # | 任务 | 文件 | 说明 |
|---|------|------|------|
| P0-D1 | SSE 前端直连 SPS | `api.ts` + `api_app.py`（加 CORS） | `EventSource("/api/v2/realtime/kline-alerts/stream")` → `EventSource("http://127.0.0.1:8090/api/v1/kline-alerts/stream")`；SPS 加 CORS 允许 localhost:5173。**认证策略**：本地开发阶段 SPS CORS 白名单 localhost:5173 即可；生产阶段使用短期 stream token，由 BFF 签发（`GET /api/v2/realtime/sse-token`），前端带 `?token=xxx` 直连 SPS |

**验收**：
```bash
# 浏览器 DevTools Network 面板：SSE 连接目标为 127.0.0.1:8090（非 BFF:8000）
```

#### P0 Commit 拆分

为了方便执行和回滚，P0 拆成 4 个独立 commit，每个 commit 可单独测试和回滚：

| Commit | 内容 | 验证 |
|--------|------|------|
| `fix(runtime-process): prevent jyhf auction and cdp zombie processes` | P0-A1 + P0-A2 | `ps aux \| grep jyhf_auction \| wc -l` = 1 |
| `fix(resource): add raw_news memory guard and phase0 cpu throttle` | P0-B1 + P0-B2 | RSS < 3GB, CPU < 40% |
| `fix(frontend): remove strict mode and merge realtime polling` | P0-C1 + P0-C2 | 2 SSE, 1 轮询 |
| `fix(sse): connect frontend directly to sps stream` | P0-D1 | SSE 目标 = 127.0.0.1:8090 |

---

### 第二层：中期架构归位（2-4 周）

**原则：引入最小新框架（Runtime Lite）、建立统一契约（Envelope Adapter）、收拢决策入口。**

#### 阶段 2：Runtime Lite（只做 5 件事，不做复杂平台）

不做：自动拉起失败服务、复杂 supervisor、Web 控制台、动态配置热更新、多机器部署。

**Runtime Lite 第一版只纳入以下服务**（先不要纳入 JYHF DOM/CDP 全量生命周期、TDX/JYHF 行情、Replay、W2S 全量服务）：

- redis / postgres（health check）
- web_app_service（8000）
- frontend_bff（8003）
- sps（8090）
- raw_news_services
- phase0_decision_services

| # | 任务 | 目录/文件 | 说明 |
|---|------|-----------|------|
| 2.1 | 新建 `runtime/` 目录 | `runtime/cli.py, orchestrator.py, health.py, runtime_state.py` | 最小 Runtime |
| 2.2 | 建立 runtime profiles | `runtime/profiles/realtime.yaml` | 声明式：列出所有 service、env、stream |
| 2.3 | 启动/停止/状态/健康检查 | `runtime/cli.py` | `python -m runtime.cli start/stop/status/health --profile realtime` |
| 2.4 | 服务启动写 runtime_service_status | 新建 DB 表 + 各服务上报 | 统一运行状态 |
| 2.5 | 建立统一 run_id | `runtime/runtime_state.py` | `run_id = f"realtime-{date}-{time}"` |
| 2.6 | `run_realtime_stack.sh` 改为调用 runtime | `scripts/run_realtime_stack.sh` | 保留兼容入口 |

**验收**：
```bash
python -m runtime.cli status
# 输出：所有服务 running/stopped/error 状态
python -m runtime.cli health
# 输出：redis: ok, postgres: ok, sps: ok, raw_news: ok, decision: ok
```

#### 阶段 3：Envelope Adapter（兼容包装，不重写上游）

**Stream 迁移四步法：Alias → 双写 → 切读 → 废弃**

当前不做全量迁移，只做 Alias 层。**新名字查旧名字**（新架构代码使用新名字，通过 alias 映射到旧 stream；所有旧代码继续读写旧 stream，不产生断链）：

```python
# 新名字 → 旧名字（第一阶段）
STREAM_ALIASES = {
    "stream:intel.raw.news": "stream:news:raw",
    "stream:intel.event.structured": "stream:events:structured",
    "stream:alert.decision": "stream:events:decision",
    "stream:ui.feed": "stream:event:feed",
}
# 旧代码继续读 stream:news:raw
# 新代码通过 alias("stream:intel.raw.news") → 实际读 stream:news:raw
```

**Envelope 兼容策略**（关键）：

```python
def ensure_envelope(message: dict, default_schema_type: str) -> dict:
    """旧消息兼容包装，不要求一次性重写所有上游。"""
    if message.get("envelope_version"):
        return message  # 已有 envelope，透传

    return {
        "envelope_version": "1.0",
        "event_id": message.get("event_id") or message.get("news_id") or generate_id(),
        "trace_id": message.get("trace_id") or generate_trace_id(),
        "run_id": current_run_id(),
        "source_type": infer_source_type(message),
        "source_name": message.get("source") or "legacy",
        "source_channel": message.get("source_channel", ""),
        "biz_date": message.get("biz_date", today()),
        "market_session": infer_session(),
        "event_time": message.get("event_time") or message.get("created_at") or now_iso(),
        "ingest_time": now_iso(),
        "schema_type": default_schema_type,
        "schema_version": "legacy-adapted",
        "payload": message,
        "quality": {},
        "routing": {},
    }
```

| # | 任务 | 文件 | 说明 |
|---|------|------|------|
| 3.1 | 新增 `core/contracts/envelope.py` | `core/contracts/envelope.py` | Envelope 定义 + `ensure_envelope()` |
| 3.2 | 新增 `core/contracts/streams.py` | `core/contracts/streams.py` | Stream alias + 配置 |
| 3.3 | RealTimeNewsCollector 输出适配 | `database_service/streams/run_realtime_news_collector.py` | 首个 native envelope 输出 |
| 3.4 | 下游 handler 兼容 envelope | `NewsStreamHandler / ThemeProcessor / DecisionExecutor` | 读 `msg.get("payload", msg)` |
| 3.5 | raw_event_log 表 + 写入 | `core/contracts/` + DB migration | 所有进入系统的原始事件落库 |

**验收**：
```bash
redis-cli --raw xrevrange stream:news:raw + - count 1
# 手动检查字段：envelope_version, event_id, trace_id, run_id, source_type, schema_type, payload
# 或用脚本：python scripts/inspect_stream_latest.py stream:news:raw
```

#### 阶段 4：Decision 收口

| # | 任务 | 说明 |
|---|------|------|
| 4.1 | 定义 SignalDecision 结构（schema_type, level, scores, evidence, risk_flags） | 统一决策输出格式 |
| 4.2 | 所有现有告警加 ensure_decision 包装 | W2S/支撑/竞价 → 统一输出 stream:alert.decision |
| 4.3 | signal_decision 表 + 落库 | 最终决策持久化 |
| 4.4 | Intel Feed 从 decision stream 读取 | 前端展示的是决策结果，非原始消息 |

**验收**：
```bash
# 任意一条 W2S 告警包含：decision_id, level, scores, evidence, risk_flags
redis-cli --raw xrevrange stream:alert.decision + - count 1
# 或用脚本：python scripts/inspect_stream_latest.py stream:alert.decision
```

#### 阶段 5：BFF 职责瘦身

**硬原则**：
1. BFF 不允许再管理长期运行进程
2. BFF 不允许再转发高频 SSE
3. BFF 不允许承载采集器生命周期

| # | 任务 | 说明 |
|---|------|------|
| 5.1 | BFF 去掉 SSE 代理 | P0-D1 已做 |
| 5.2 | CDP/auction 生命周期管理移到 runtime | Runtime orchestrator 统一管理进程 |
| 5.3 | BFF 只保留：API 聚合、认证鉴权、轻量查询、SPA 服务 | 瘦身为前端查询网关 |

---

### 第三层：长期平台化能力（4-8 周）

**原则：先 facade 封装旧服务，再逐步内聚业务逻辑。不重写，只归位。**

#### 阶段 6：领域引擎 Facade

关键原则：**engines/ 第一阶段不重写业务逻辑，只做 facade 封装。现有 service 先被 engines 调用。等接口稳定后，再逐步把核心逻辑内聚到 engines。**

```python
class W2SEngine:
    """Facade: 包装现有 W2SUnifiedAlertService，不重写逻辑"""
    def __init__(self):
        self.legacy = W2SUnifiedAlertService()

    async def evaluate(self, market_state, support_signal, theme_state):
        return await self.legacy.score(...)
```

| # | 任务 | 说明 |
|---|------|------|
| 6.1 | EventEngine facade | 包装 NewsStreamHandler + NewsStreamProcessor |
| 6.2 | ThemeEngine facade | 包装 ThemeProcessor，保持"不自动建题材"原则 |
| 6.3 | MarketStateEngine facade | 包装现有 minute_state_builder |
| 6.4 | SupportEngine facade | 包装 kline_break_detector |
| 6.5 | W2SEngine facade | 包装 w2s_unified_alert_service |
| 6.6 | DecisionEngine facade | 消费所有 signal，输出统一 SignalDecision |

#### 阶段 7：Source Adapter 迁移（逐个迁移，先新闻后行情）

| # | 任务 | 说明 |
|---|------|------|
| 7.1 | BaseSourceAdapter 接口 | `fetch() / normalize_minimal() / publish()` |
| 7.2 | 只迁移 RealTimeNewsCollector → NewsSourceAdapter | 第一个最小闭环 |
| 7.3 | 验证 stream:intel.raw.news 正常后，再迁移 JYHF DOM | 第二条 |
| 7.4 | 最后迁移 TDX/JYHF 行情 | 行情链更脆弱，最后动 |

#### 阶段 8：报告与回放

| # | 任务 | 说明 |
|---|------|------|
| 8.1 | PremarketBriefing 消费 decision stream | `apps/premarket_briefing/` |
| 8.2 | PostmarketRecap 消费当天 decision + market close | `apps/postmarket_recap/` |
| 8.3 | ReplaySourceAdapter | `source_adapters/replay/`，按时间顺序回放历史数据 |
| 8.4 | EvaluationService | 消费 SignalDecision，输出命中率/误报率/漏报案例 |

---

## 5. BFF 职责拆分方案

### 现状

BFF（web_app_service, port 8000）承担：
- SPA 静态文件服务
- `/api/v2/*` 全部 API 路由
- JyhfCdpManager（CDP 进程生命周期）
- JyhfAuctionManager（竞价采集器生命周期）
- RealtimeStackManager（新链实时栈启动/停止）
- SSE 代理（K-line/W2S alert streams → SPS）
- JWT 认证中间件
- 9:10 自动启动采集器守护任务

### 目标拆分

| 职责 | 现状 | 目标 |
|------|------|------|
| SPA 静态文件 | BFF | BFF 保留 |
| API 聚合/查询 | BFF | BFF 保留 |
| JWT 认证 | BFF | BFF 保留 |
| CDP 进程管理 | BFF in-process | Runtime orchestrator |
| 竞价采集器管理 | BFF in-process | Runtime orchestrator |
| 新链栈管理 | BFF in-process | Runtime orchestrator |
| SSE 代理 | BFF proxy → SPS | 前端直连 SPS |
| 9:10 自动启动 | BFF background task | Runtime scheduler |

BFF 从"万能中控"降级为"前端查询网关"。

---

## 6. 统一 Envelope 设计

```json
{
  "envelope_version": "1.0",
  "event_id": "evt_20260528_xxxxx",
  "trace_id": "trace_20260528_xxxxx",
  "run_id": "realtime-20260528-093000",
  "source_type": "news",
  "source_name": "akshare",
  "source_channel": "cls",
  "collector_name": "RealTimeNewsCollector",
  "collector_version": "phase4e",
  "biz_date": "2026-05-28",
  "market_session": "intraday",
  "event_time": "2026-05-28T09:45:01",
  "ingest_time": "2026-05-28T09:45:03",
  "schema_type": "NewsRawEvent",
  "schema_version": "1.0",
  "payload": {},
  "quality": {
    "prefilter_decision": "PASS",
    "dedup_key": "sha1_xxx",
    "dedup_status": "new"
  },
  "routing": {
    "input_stream": null,
    "output_stream": "stream:intel.raw.news",
    "retry_count": 0
  }
}
```

### 必须保留的追踪字段

| 字段 | 说明 |
|------|------|
| event_id | 当前业务事件 ID |
| trace_id | 全链路追踪 ID |
| run_id | 本轮系统启动 ID |
| source_type | news/dom/market/signal/alert/report |
| source_name | akshare/jyhf/tdx/cls/manual/replay |
| source_channel | 细分来源 |
| biz_date | 交易日 |
| market_session | premarket/auction/morning/intraday/afternoon/close/postmarket |
| schema_type | payload 类型 |
| schema_version | payload 版本 |
| created_at | 产生时间 |
| ingest_time | 入系统时间 |

---

## 7. Stream 命名统一

```text
stream:intel.raw.news        ← stream:news:raw
stream:intel.raw.dom
stream:intel.event.structured ← stream:events:structured
stream:theme.match
stream:theme.state

stream:market.tick
stream:market.auction
stream:market.minute
stream:market.kline
stream:market.state

stream:signal.support
stream:signal.w2s
stream:signal.auction
stream:signal.tail

stream:alert.observation
stream:alert.candidate
stream:alert.decision       ← stream:events:decision

stream:ui.feed              ← stream:event:feed
stream:report.premarket
stream:report.intraday
stream:report.postmarket

stream:dlq
```

### Stream 迁移四步法

```
第一步 Alias：新名字 → 映射到旧名字（旧代码继续读旧 stream）
第二步 双写：核心 producer 同时写新旧 stream，验证下游可消费
第三步 切读：下游逐步改读新 stream
第四步 废弃：移除旧 stream
```

---

## 8. 告警分层

```
L0 noise       噪音，不展示
L1 observation 观察信号，进入后台/日志
L2 watch       观察池，前端弱提示
L3 alert       告警，进入 Intel 页面
L4 decision    重点决策，进入盘前必读/盘中置顶
```

---

## 9. SignalDecision 结构

```json
{
  "schema_type": "SignalDecision",
  "payload": {
    "decision_id": "dec_xxx",
    "decision_type": "w2s_watch",
    "level": "alert",
    "stock_code": "002361.SZ",
    "stock_name": "神剑股份",
    "theme_id": "xxx",
    "theme_name": "商业航天",
    "title": "神剑股份出现弱转强观察信号",
    "summary": "该股回踩缺口支撑后修复，盘中站上 VWAP，相对强度提升...",
    "scores": {
      "final_score": 81,
      "w2s_score": 78,
      "support_score": 86,
      "theme_score": 80,
      "risk_score": 32
    },
    "evidence": [
      {"type": "support", "text": "回踩缺口支撑未破"},
      {"type": "market", "text": "站上 VWAP 且相对强于板块"},
      {"type": "theme", "text": "商业航天题材近期持续活跃"}
    ],
    "risk_flags": [
      "若跌破支撑位则信号失效",
      "若成交额不能继续放大则降级"
    ],
    "suggested_action": "watch",
    "expires_at": "2026-05-28T15:00:00"
  }
}
```

---

## 10. 数据库表（分批落地）

> **P0 阶段不新增任何数据库表。P0 只修复稳定性问题。**

### 第一批 P1（中期架构归位时建表）

| 表 | 用途 | 说明 |
|----|------|------|
| runtime_service_status | 服务运行态（run_id, service_name, status, pid, last_heartbeat） | Runtime Lite 首次启动时建表 |
| raw_event_log | 所有原始事件 envelope 落库（event_id, trace_id, source_type, payload） | **只落 news/dom/signal/decision，不落 market.tick 高频行情。** tick 不进 raw_event_log，只在必要时落 market_state_snapshot 或抽样审计，防止 PostgreSQL 快速膨胀 |
| signal_decision | 最终决策落库（decision_id, level, scores, evidence, risk_flags） | Decision 收口阶段建表 |

### 第二批 P2（后续）

| 表 | 用途 | 说明 |
|----|------|------|
| signal_observation | 观察信号 | 数据量大，先存 Redis/日志；等 decision 层稳定后再落库 |
| market_state_snapshot | 分钟状态快照 | 数据量极大，需精心设计分区和保留策略，防膨胀 |

---

## 11. 前端定位

| 页面 | 定位 | 面向 | 内容 |
|------|------|------|------|
| 实时采集页面 | 运维控制台 | 开发者/运维 | 服务状态、采集器状态、Stream 积压、Consumer Group lag、DLQ 数量、SSE 连接数、最近错误、进程 PID |
| Intel 页面 | 投资情报台 | 使用者 | 重要事件、题材驱动、支撑告警、W2S 观察、综合决策、风险提示 |
| 盘前必读 | 决策报告 | 使用者 | 今日核心事件、主线题材、W2S 观察池、支撑观察池、风险提示 |
| 盘后复盘 | 归因报告 | 使用者 | 今日提示命中率、误报案例、漏报案例、明日重点 |

---

## 12. 目录结构

```text
alpha_pilot/
  core/
    contracts/
      envelope.py
      streams.py
      schemas.py

  runtime/
    cli.py
    orchestrator.py
    health.py
    runtime_state.py
    config_loader.py
    profiles/
      realtime.yaml

  source_adapters/
    base.py
    news/
    jyhf/
    market/
    replay/

  engines/
    event_engine/
    theme_engine/
    market_state_engine/
    support_engine/
    w2s_engine/
    decision_engine/

  apps/
    intel_feed/
    premarket_briefing/
    postmarket_recap/

  docs/architecture/
```

---

## 13. 暂不做事项

1. 暂不引入 Kafka，Redis Stream 继续够用
2. 暂不拆成 Docker 微服务，先用 Runtime Lite 管理本地多进程
3. 暂不重写 ThemeProcessor / DecisionExecutor，先做 envelope 兼容包装
4. 暂不全面迁移所有 Source Adapter，先只迁移新闻采集（RealTimeNewsCollector）
5. 暂不把所有观察信号落库，先只落 final decision
6. 暂不做自动交易，只做观察、告警、复盘
7. 暂不做复杂权限系统，BFF 先做职责瘦身
8. 暂不做 Web 控制台、动态配置热更新、多机器部署

---

## 14. 全链路验收

### P0 止血验收

```bash
# 进程治理
ps aux | grep jyhf_auction | grep -v grep | wc -l         # = 1
curl -s http://127.0.0.1:8000/api/v2/realtime/jyhf-cdp/status | python -c "import sys,json; print(json.load(sys.stdin).get('service_owner'))"

# 资源治理
ps aux | grep raw_news | awk '{print int($6/1024)}'         # RSS < 3072 MB
ps aux | grep phase0 | awk '{print $3}'                     # CPU < 40%

# 连接治理
# Browser DevTools Network: SSE = 2 个（非 4 个）
# Browser DevTools Network: 状态轮询 = 1 个（非 3 个）

# SSE 瘦身
# Browser DevTools Network: SSE 目标 = 127.0.0.1:8090（非 BFF:8000）
```

### Runtime Lite 验收

```bash
python -m runtime.cli status
python -m runtime.cli health
python -m runtime.cli start --profile realtime
python -m runtime.cli stop --profile realtime
```

### Envelope 验收

```bash
redis-cli --raw xrevrange stream:news:raw + - count 1
# 手动检查字段：envelope_version, event_id, trace_id, run_id, source_type, schema_type, payload
```

### Decision 收口验收

```bash
redis-cli --raw xrevrange stream:alert.decision + - count 1
# 手动检查字段：decision_id, level, scores, evidence, risk_flags
```

### 全链路追踪验收

```bash
# 任意一条 Intel Feed item 能追溯到 raw event：
# feed item → trace_id → raw_event_log → 原始采集事件
```
