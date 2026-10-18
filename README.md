# 对自己插件的卸载规范 / Uninstall Specification for My Own Plugins

> **⚠️ 声明（必读）/ Disclaimer (must read)**
>
> 本仓库是 **Rad-Weed** 给**自己的 AI（辅助制作 DSH 插件时）**设定的**约束规则**：AI 在制作、封装、安装与卸载 DSH 插件时必须遵守。
> 约束对象是**我自己的 AI 的插件制作行为**；它不是 DSH 官方规范，也不是任何插件市场的准入条件或评审标准。
> 其他作者可以自由参考、引用、裁剪，无需授权；是否采纳完全自愿。
>
> This repository defines **rules that Rad-Weed imposes on their own AI assistant when building DSH plugins**: the AI must follow them while creating, packaging, installing, and uninstalling plugins.
> The rules apply to **my own AI's plugin-development behavior**; they are not an official DSH standard, nor a marketplace admission condition or review criterion.
> Other authors may freely reference, quote, or adapt it without permission; adoption is entirely voluntary.

---

## 📖 全文 / Full text

| 语言 / Language | 文件 / File |
|---|---|
| 中文（简体） | [docs/zh/SPEC.md](docs/zh/SPEC.md) |
| English | [docs/en/SPEC.md](docs/en/SPEC.md) |

**版本 / Version**: v1.0（首个版本 / First version） · 2026-10-19

---

## 这是什么 / What this is

一份由我给**自己的 AI** 制定的、针对「插件安装、卸载、外部副作用、中断恢复」的约束规范，缘于 2026-10-19 一次真实的 DSH 插件卸载事故：

- 官方卸载路径因 pnpm registry 重试静默耗时 **4 分 10 秒**；
- 移除窗口期出现误导性的 `not-removable`；
- 卸载成功后 `node_modules` 里残留目录联接（junction）；
- 桌面壳补丁（`app.asar`）必须等 DSH 完全退出才能还原，期间两次中断留下半状态。

This is a specification I wrote to constrain **my own AI** on plugin **installation, uninstallation, external side effects, and interruption recovery**, after a real DSH plugin uninstallation incident on 2026-10-19:

- the official removal path was silently stuck for **4 min 10 s** due to pnpm registry retries;
- a misleading `not-removable` appeared during the removal window;
- stale directory junctions were left in `node_modules` after a successful uninstall;
- the desktop shell patch (`app.asar`) could only be reverted after DSH fully exited, and two interruptions left a half-applied state.

## 里面有什么 / What's inside

- **§0 声明**：本规范的约束对象是我自己的 AI；
- **TL;DR**：我对自己插件的 10 条；
- **§1 事故复盘**：可复现的证据时间线；
- **§2 生态规范盘点**：官方 / 市场 / 验证 / 社区规范索引，以及它们未覆盖的缺口；
- **§3 AI 执行条款**：6 条强制（M）、4 条推荐（R）、反模式清单（A）；
- **§4 参考实现**：幂等卸载脚本骨架（独立进程等待退出、双 SHA 校验、失败保留现场、自动重启）；
- **§5 自测清单**：提交/发版前逐项检查；
- **§6 生态观察**：仅供个人记录，不构成对任何人的要求；
- **§7 参考来源** + **附录 事故时间线**。

## 适用范围 / Scope

- ✅ 我自己的 AI 在为我制作/维护 DSH 插件时的操作行为（尤其是安装脚本、壳补丁、外部文件/系统级改动）；
- ℹ️ 仅描述我对自己 AI 的要求，不针对任何第三方插件、作者或项目作出评价；
- ℹ️ 本仓库为个人文档，与 DeepSeek 官方、DSH 插件市场及评审/验证机构无关。

## 署名与授权 / Attribution & Authorization

- 修改出版，请联系我加入你们学习。
- 文中事故复盘仅代表个人经验，不针对任何具体插件、作者或项目。

*For modification or publication, please contact me so I can join you in learning.*

---

*本文档由 AI 与本人调研部分文档后生成，不一定全面。*
