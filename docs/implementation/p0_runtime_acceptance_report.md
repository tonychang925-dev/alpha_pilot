# P0 运行态验收报告

日期：2026-05-28  
项目：ai_theme_app legacy 系统  
目标：验证 P0-A/P0-B 稳定性修复是否生效

## 验收结果

| 验收项 | 状态 | 详情 |
|--------|------|------|
| P0-A1 竞价进程不重复 | 通过 | 连续 3 次启动后仅 1 个进程，子进程带 `--parent-pid` |
| P0-A2 CDP pidfile 恢复 | 通过 | BFF 重启后从 pidfile 恢复 `owner=managed, pid=25747` |
| P0-B1 raw_news 内存保护 | 初步通过 | 新进程已加载 watchdog，默认超过 3GB `exit(137)` |
| P0-B2 phase0 CPU 限流 | 初步通过 | `batch_delay_ms=100` 已加载 |

## 改动文件清单

| 文件 | 改动 |
|------|------|
| `web_app_service/services/jyhf_auction_manager.py` | start() 前 poll() 检查；stop() 前 os.kill(pid,0) 验证；子进程传 `--parent-pid` |
| `stock_processing_service/collectors/jyhf_auction_collector.py` | 新增 `--parent-pid` CLI 参数 + 10s watchdog |
| `web_app_service/services/jyhf_cdp_manager.py` | `__init__` 新增 pidfile 恢复；launch 后写 pidfile；stop 后删 pidfile |
| `evaluate_service/.../run_raw_news_services.py` | 新增 `_watch_memory()`: 30s RSS 监控，超限 `sys.exit(137)` |
| `database_service/streams/handlers/theme_processor.py` | batch 间 `await asyncio.sleep(batch_delay_ms/1000)` 可配置 |
| `evaluate_service/.../run_phase0_decision_services.py` | 新增 `PHASE0_BATCH_DELAY_MS` 环境变量 |

## 额外发现

验收过程中发现历史遗留孤儿进程 52604/52605：

- 52604（raw_news）：占用约 2.6GB 内存和 400% CPU
- 52605（phase0）：已脱离父进程 SPS

已手动清理。该问题验证了 P0-A 进程治理和 P0-B 资源治理的必要性。

## 后续观察

raw_news 和 phase0 新进程（PID 59316/59317）刚启动，模型尚未完全加载，仍需持续观察 30~60 分钟：

- raw_news RSS 是否持续逼近 3GB
- phase0 CPU 是否长期超过 40%
- 是否再次出现孤儿进程
- 是否频繁 `exit(137)`

## 结论

P0-A/P0-B 代码已就位，静态验收和初步运行态验收通过。完成持续观察后，可进入 P0-C：前端连接治理。
