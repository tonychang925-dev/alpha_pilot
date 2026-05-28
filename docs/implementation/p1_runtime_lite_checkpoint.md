# P1-1 Runtime Lite 验收

日期：2026-05-28  
分支：`p1/runtime-lite-status`（ai_theme_app）  
状态：已提交，待 push/PR/合并

## 交付内容

| 文件 | 说明 |
|------|------|
| `runtime/__init__.py` | 包初始化 |
| `runtime/cli.py` | CLI 入口：`status` / `health` 命令 |
| `runtime/health.py` | 健康检查引擎：TCP / HTTP / process 三种探针 |
| `runtime/profiles/realtime.yaml` | 服务注册表：8 个服务声明 |

## 验收结果

```
$ python -m runtime.cli status --profile realtime
============================================================
  🟢 redis (必)
  🟢 postgres (必)
  🟢 web_app_service (必)
  🟢 sps (必)
  🟢 jyhf_cdp_service
  🟢 raw_news_services
  🟢 phase0_decision_services
  🟢 frontend_dev
============================================================
Running: 8/8
```

```
$ python -m runtime.cli health --profile realtime
Profile: realtime
Status:  ok
--------------------------------------------------
  [OK] redis (127.0.0.1:6379)
  [OK] postgres (127.0.0.1:5432)
  [OK] web_app_service (HTTP 200)
  [OK] sps (HTTP 200)
  [OK] jyhf_cdp_service (HTTP 200)
  [OK] raw_news_services (pids=77244)
  [OK] phase0_decision_services (pids=77245)
  [OK] frontend_dev (HTTP 200)
```

## 设计决策

- **只做 status/health，不做 start/stop**：避免过早接管进程生命周期
- **YAML 声明式配置**：服务列表与检查逻辑分离
- **零新依赖**：仅用项目已有的 httpx + yaml

## 下一步

P1-2：Envelope Adapter — 统一事件契约兼容层
