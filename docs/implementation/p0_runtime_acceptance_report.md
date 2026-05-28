# P0 运行态验收报告（最终版）

日期：2026-05-28  
项目：ai_theme_app legacy 系统  
目标：验证 P0-A/B/C/D 稳定性修复全部生效

## 验收结果

| 验收项 | 状态 | 分支 | 详情 |
|--------|------|------|------|
| P0-A1 竞价进程不重复 | ✅ 通过 | main | 连续 3 次启动仅 1 进程，`--parent-pid` 生效 |
| P0-A2 CDP pidfile 恢复 | ✅ 通过 | main | BFF 重启后 pidfile 恢复 `owner=managed, pid=25747` |
| P0-B1 raw_news 内存保护 | ✅ 通过 | main | 新进程已加载 watchdog，30s RSS 监控，超 3GB exit(137) |
| P0-B2 phase0 CPU 限流 | ✅ 通过 | main | `batch_delay_ms=100` 已加载，稳态 CPU 0.0%-0.3% |
| P0-C1 React StrictMode 移除 | ✅ 通过 | main (PR #318) | 消除 SSE 双连接（4→2） |
| P0-C2 轮询合并 | ✅ 通过 | main (PR #318) | `/api/v2/realtime/status-bundle` 替代 3 个独立轮询 |
| P0-D SSE 直连 SPS | ✅ 通过 | `fix/sse-direct-to-sps` | 前端 EventSource 直连 SPS:8090，去掉 BFF 代理 |

## 改动文件清单

| 文件 | 阶段 | 改动 |
|------|------|------|
| `web_app_service/services/jyhf_auction_manager.py` | P0-A1 | start() 前 poll()；stop() 前 PID 验证；`--parent-pid` |
| `stock_processing_service/collectors/jyhf_auction_collector.py` | P0-A1 | `--parent-pid` CLI + 10s watchdog |
| `web_app_service/services/jyhf_cdp_manager.py` | P0-A2 | pidfile 写/读/删，`__init__` 恢复 ownership |
| `evaluate_service/.../run_raw_news_services.py` | P0-B1 | `_watch_memory()`: 30s RSS 监控 → exit(137) |
| `database_service/streams/handlers/theme_processor.py` | P0-B2 | `batch_delay_ms` 可配置 |
| `evaluate_service/.../run_phase0_decision_services.py` | P0-B2 | `PHASE0_BATCH_DELAY_MS` env |
| `frontend/src/main.tsx` | P0-C1 | 移除 `<React.StrictMode>` |
| `web_app_service/api/routes.py` | P0-C2 | 新增 `/api/v2/realtime/status-bundle` |
| `frontend/src/lib/api.ts` | P0-C2/P0-D | `fetchStatusBundle()` + SSE URL 改直连 SPS |
| `frontend/src/routes/collection/RealtimeCollectorPage.tsx` | P0-C2 | 3→1 轮询合并 + scope hotfix |
| `stock_processing_service/api_app.py` | P0-D | CORS 中间件 |

## 额外发现与处理

1. 验收过程中发现历史遗留孤儿进程 52604/52605（2.6GB/400%CPU），已手动清理
2. CDP Intel push 未启用的根因：CDP 被手动启动缺乏 `JYHF_CDP_PUSH_INTEL=1`，通过 BFF 重启后修复
3. W2S 告警实时生成服务不存在（仅手动回放脚本）→ 归入第二阶段"领域引擎层"

## 已知遗留

- W2S 弱转强告警没有实时生成后台服务，Redis 中仅有历史回放数据
- KlineBreakDetector 在下午时段可能无新告警生成

## 结论

P0 止血层全部验收通过。系统状态从"进程泄漏、资源失控、连接风暴、SSE 冗余"恢复到可控状态。
