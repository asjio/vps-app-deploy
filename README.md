# vps-app-deploy

VPS 应用部署任务书 + 全部资源(图标/PWA模板/通知代码)。

**给家里 AI 的唯一入口: 读 [DEPLOY.md](DEPLOY.md)**, 里面有完整任务、步骤、资源路径和验收标准。

## 仓库结构

```
DEPLOY.md          <- 任务书(自包含, 照做即可)
icons/             <- 门户/NextDay 的 APP 图标(192/512两档)
pwa/sw.js          <- Service Worker 模板
code/notifier.py   <- Server酱微信通知模块
```

## 背景

- 趋势漏斗: github.com/asjio/trend-funnel (已含移动端卡片化+PWA, 直接部署)
- NextDay 台账: github.com/asjio/nextday (需按 DEPLOY.md 做移动端适配)
- 目标 VPS: 107.172.208.174
