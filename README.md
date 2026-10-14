# Engineering Notes · 真实工程问题排查笔记

记录实际开发、AI 编程工具和网络环境中的 **真实现象 → 排查过程 → Root Cause → 修复方案 → 验证与工程经验**。

这些文章以实际复现和解决经历为基础；推测性根因会明确标注，方便他人复现、验证和迁移。

## 排障案例

| 日期 | 文章 | 关键结论 |
| --- | --- | --- |
| 2026-10-14 | [Windows Codex Desktop 反复 Reconnecting：系统代理正常但持续重连](troubleshooting/codex-desktop-windows-proxy-reconnecting.md) | 显式设置 Codex 的 HTTP_PROXY / HTTPS_PROXY 后，重连故障在实测中消失 |

## 内容方向

- AI 编程工具 / Codex / Agent 实践
- 全栈开发中的真实工程问题
- 网络、代理、部署与开发环境排障
- Root Cause 分析及跨项目迁移

> 所有解决方案请结合软件版本和实际环境验证，不应直接覆盖个人配置文件。
