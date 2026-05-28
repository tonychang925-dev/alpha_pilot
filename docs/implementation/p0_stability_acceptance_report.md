# P0 稳定性修复验收报告

日期：2026-05-28  
Legacy 项目：ai_theme_app  
新架构项目：AlphaPilot / alpha_pilot

## 验收结论

P0 止血阶段全部完成并通过运行态验收。

| 阶段 | 内容 | 状态 |
|------|------|------|
| P0-A1 | 竞价进程治理，防重复启动 | ✅ 通过 |
| P0-A2 | CDP pidfile 恢复，BFF 重启后 ownership 不丢 | ✅ 通过 |
| P0-B1 | raw_news RSS 内存保护 | ✅ 通过 |
| P0-B2 | phase0 CPU batch delay 限流 | ✅ 通过 |
| P0-C1 | 移除 React StrictMode，减少 dev 双挂载 | ✅ 通过 |
| P0-C2 | 合并实时采集页状态轮询 | ✅ 通过 |
| P0-D | SSE 前端直连 SPS，绕开 BFF 代理 | ✅ 通过 |

## 关键验收结果

- 竞价采集器连续多次启动后仅保留 1 个进程
- CDP pidfile 可恢复 owner=managed
- raw_news RSS 稳定在 3GB 阈值以内
- phase0 CPU 稳态接近 0%
- K线告警 SSE 正常连接，数据正常流入
- W2S 告警 SSE 正常连接，数据正常流入
- 无 CORS 报错
- CDP 正常采集，capture ok events=14 new=14 token=yes
- 实时采集页面负责展示 CDP 运行状态和日志，具体 CDP 事件进入 Intel Feed 页面

## 结论

P0 阶段完成了生产稳定性止血。  
下一阶段可进入 P1：Runtime Lite + Envelope Adapter + Decision 收口。
