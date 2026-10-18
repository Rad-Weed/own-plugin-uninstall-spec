# 对自己插件的卸载规范（v1.0 · 首个版本）：我要求自己的 AI 在制作插件时遵守的安装、卸载与外部副作用约束

> **适用范围**：**本人（Rad-Weed）给自己的 AI（辅助制作 DSH 插件时）设定的约束规则**。AI 在制作、封装、安装与卸载 DSH 插件时必须遵守，尤其是带安装脚本、桌面壳补丁、外部文件/系统级改动的插件。
> **缘起**：2026-10-19 一次真实卸载事故（`dsh-whale-widget` 与 `dsh-restart-button`）：官方卸载路径耗时 4 分 10 秒、出现误导性 `not-removable`、残留 `node_modules` 目录联接、壳补丁必须等 DSH 完全退出才能还原。
> **定位**：这是**我给自己 AI 的插件制作约束清单**，约束对象是我自己的 AI 在插件开发中的行为；它不是 DSH 官方规范；其他作者可自由参考、引用、裁剪。官方与市场已有规范见 §2，本文只补它们未覆盖的「卸载、回滚、外部副作用、中断恢复」部分。
> **版本**：v1.0（首个版本）· 2026-10-19

---

## 0. 声明（必读）

- 本规范**面向我自己的 AI**：我要求它在为我制作、封装、安装与卸载 DSH 插件时遵循这些规则。
- 约束对象是**我自己的 AI 的插件制作行为**；本规范不是 DSH 官方规范，也不构成插件市场准入条件或评审标准。
- 其他作者可以自由参考、引用、裁剪，无需授权；是否采纳完全自愿。
- 文中的事故复盘与条款仅代表我的个人经验，不针对任何插件、作者或项目。

## TL;DR（我要求自己的 AI 遵守的 10 条）

1. **安装与卸载必须对称**：每一个安装动作都要有一个可重入、幂等的反向动作；卸载脚本连跑两次必须无副作用。
2. **外部副作用必须声明**：改了 `app.asar`、系统配置、桌面壳、服务或用户目录之外的任何东西，都要在 README / `package.json` 里写明改了什么、怎么还原。
3. **壳补丁类插件的安装/卸载必须带编排**：检测 DSH 是否运行 → 等待完全退出 → 改/还原文件 → 校验 SHA256 → 重启；**绝不在 DSH 运行时写 `app.asar`**。
4. **等退出必须用独立进程**：DSH 退出会同步终止子进程树，等待脚本要用 WMI / 计划任务“逃逸”，不要挂 DSH 的普通子进程。
5. **卸载后必须自检并清理 DSH 四件套**：`package.json` 的 dependencies/bundles、`pnpm-lock.yaml`、`node_modules/.modules.yaml`、`cordis.yml`，以及 `node_modules/<pkg>` 残留联接。
6. **用户数据默认保留**：卸载默认不动用户数据；清除数据必须显式选项（`--purge`）并在 README 写清路径。
7. **卸载要有耗时预期与日志**：DSH 的 remove 可能因 pnpm 网络重试静默数分钟（实测 4m10s），脚本必须写日志、给超时、别让人以为卡死。
8. **容器内设计优先**：能用 Cordis 宿主能力/服务/插槽解决的，不要碰安装目录；`ctx.effect` 兜底所有资源清理，无 daemon、无裸子进程。
9. **依赖与兼容性按官方契约**：宿主接口包只进 `peerDependencies`；声明 `engines.dsh`；版本必 bump；入口/导出必须真实存在。
10. **安全与披露按市场契约**：`cloud/network/api_keys/permissions` 如实声明；不裸 `child_process`（P202）、不抢 single 槽（P401）。

---

## 1. 为什么我要给 AI 立这份规范（实测事故摘要）

| 现象 | 实测证据 | 我自己可规避的部分 |
|---|---|---|
| 卸载像“卡死” | `pnpm remove` 触发 registry 网络重试（10s → 1min 退避），两次卸载均 `Done in 4m 10.6s`，全程无输出 | 卸载脚本加日志与耗时提示；不要依赖联网的卸载路径 |
| 误导性错误 | pnpm 刚写完 `package.json`、bundle 已从列表消失的窗口内，再次移除返回 `not-removable`（无诊断） | 文档提示“等上一步完全结束再重试”；先 disable 再 remove |
| 残留目录联接 | 卸载成功后 `node_modules/dsh-whale-widget`、`node_modules/dsh-restart-button` junction 仍在 | 卸载脚本末尾自检并清理（见 §3.4） |
| 壳补丁无法在线还原 | `app.asar` 被运行中的 DSH 独占锁定，必须完全退出 | 安装/卸载脚本自带“退出→应用→重启”编排（见 §3.3、§4） |
| 中断后状态不一致 | 用户两次关闭卸载窗口 → pnpm 中断 → 三次重叠的 remove 操作，配置与依赖半同步 | 幂等 + 可重入 + 失败可重试 + 明确恢复指引 |
| 恢复需要人工介入 | 最终由独立后台脚本：等退出 → 官方 restore → SHA 校验 → 删备份 → 自动重启，耗时 4 秒完成 | 把这套编排内置进插件自己的安装/卸载脚本 |

> 结论：**插件管理器只负责“Cordis 行 + pnpm 依赖”这一层**；任何超出该层的东西（壳补丁、外部文件、系统设置）都不在它的卸载契约里，必须由插件作者自己负责“对称清理”。

---

## 2. 现有规范盘点（生态里有什么）

| 文档 | 覆盖层 | 关键内容 | 与本规范的关系 |
|---|---|---|---|
| 官方《打包与安装插件》[publish.zh.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.zh.md) | 打包 / 安装契约 | bundle 与 profile 两个 manifest、`dsh.bundle.patch`、`engines.dsh`、peer 依赖、加载顺序、`dsh plugin remove` 会同时移除依赖与层；Git 安装的 `prepare` + `allowBuilds` 授权 | **安装侧的官方权威**；未涉及外部副作用清理 |
| 官方《插件与生命周期》[framework/index.zh.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/index.zh.md) | 插件运行时生命周期 | Fiber、`ctx.effect`、disposer 逆序、HMR/reload | 运行时资源清理的官方依据（§3.9） |
| DSH 插件市场 [STANDARD.md](https://github.com/bradeGithub/DSH-Plugins-Marketplace/blob/main/STANDARD.md) | 识别 / 安装 / 披露 / 验证 | 10 步类型判定、cordis 插件最小 `package.json`、反模式（根目录 install 脚本、自己注册 patch、宿主包进 dependencies…）、DISCLOSURE v0.2、验证层字段契约、自测清单 | **市场“识别层”权威**；其卸载速查仅一句“脚本型不可回滚”，无卸载契约 |
| [dsh-plugin-verify](https://github.com/qing3a/dsh-plugin-verify) | 运行时验证 / 评审 | 7/7 waterfall + tools/result 运行时判定、P202（裸子进程）/P401（single 槽）安全规则、[plugin-standards.md](https://github.com/qing3a/dsh-plugin-verify/blob/main/docs/plugin-standards.md)、[review-checklist.md](https://github.com/qing3a/dsh-plugin-verify/blob/main/docs/review-checklist.md)；收录条件已要求 README 含**安装/卸载**说明 | **验证层事实标准**；可把本文 §5 作为其补充 |
| [dsh-design-conventions.md](https://github.com/qing3a/dsh-plugin-verify/blob/main/docs/source/dsh-design-conventions.md) | 官方设计规范整理 | 无 daemon、host 退出 SIGKILL 全部子进程树、`ctx.subprocess`、`ctx.storageDomain`、slot 规则、错误码契约、i18n/文档规范 | 本文 §3.9 的依据 |
| [dsh-agent-teams 的 dsh-plugin-development Skill](https://github.com/NanmiCoder/dsh-agent-teams/blob/main/.dsh/skills/dsh-plugin-development/SKILL.md) | 开发执行清单 | 运行面判断、bundle 契约、effect 所有权、client 构建、HMR/重启边界、验证矩阵、完成标准 | 实现层面的最佳实践 |
| [dshbase](https://dshbase.com/) / [awesome-dsh-plugin](https://github.com/awesome-dsh-plugin/awesome-dsh-plugin) | 目录 / 分级 | 收录门槛、install-tested 徽标、安全免责声明 | 分发渠道侧要求 |

**调研结论**：

- 生态**已经有**“怎么写才能被识别/安装/更新”的规范（市场 STANDARD）、
- **已经有**“能不能信”的验证规范（dsh-plugin-verify、披露契约）、
- **也已经有**“怎么写才不烂”的开发规范（官方文档 + 社区 Skill），
- **但没有任何一份**把「**卸载、回滚、外部副作用、中断恢复**」写成作者契约。市场文档甚至明确写了脚本型安装“无法自动回滚，需作者提供反向操作说明”——只有一句话，没有可执行要求。

本文档 §3 就是我把这一块补成的可执行最小契约，作为我要求自己的 AI 在插件制作中执行的标准。

---

## 3. 我要求自己的 AI 遵守的执行条款（M=强制 · R=推荐 · A=反模式）

### 3.1 ⛔ M1 —— 安装与卸载必须对称、幂等、可重入

- 每个 install 动作必须有对应 uninstall；脚本连跑两次必须无副作用（第二次输出“已是目标状态”并 exit 0）。
- 卸载脚本必须能在**任意中间状态**启动：依赖已删/未删、壳补丁已还原/未还原、备份已删/未删，都要能安全收敛。
- 卸载失败必须**可重试**，并在失败时保留现场（备份、日志、必要文件），而不是把状态改到一半。
- 提供 `-DryRun` / `--dry-run`：只打印将做什么，不改任何文件。

### 3.2 ⛔ M2 —— 外部副作用必须声明并给出反向操作

在 README（推荐同时放 `package.json` 的自定义字段）中列出：

| 声明项 | 说明 |
|---|---|
| 改了哪些文件 | 绝对含义而非绝对路径，如 `DSH 安装目录/resources/app.asar` |
| 改了什么 | 壳补丁 / 系统托盘 / 文件关联 / 计划任务 / 环境变量 / 端口 |
| 是否可在线还原 | 若否，写明“需要 DSH 完全退出后还原” |
| 备份位置与校验 | 备份目录、SHA256、原始大小 |
| 反向操作 | 卸载命令 / 脚本路径 / 手动步骤 |
| 用户数据位置 | 数据目录、是否默认保留、如何清除 |

**建议的 manifest 草案字段**（供市场/宿主未来消费，当前写进 README 也有效）：

```jsonc
{
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "install": {
      "effects": ["app.asar"],            // app.asar | profile | user-data | system | service
      "requiresFullExit": true,           // 卸载是否必须 DSH 完全退出
      "uninstall": "./scripts/uninstall.ps1",
      "backup": { "dir": "resources/.dsh-myplugin-backup", "sha256": true }
    },
    "data": { "path": "~/.dsh/my-plugin", "purgeFlag": "--purge" }
  }
}
```

### 3.3 ⛔ M3 —— 壳补丁 / 系统级改动专项契约

- **绝不在 DSH 运行时写 `app.asar`**：运行中文件被独占锁定，截断会把安装写坏。
- 安装前必须**先备份**：原始 header + `meta.json`（`originalSize`、`originalSha256`、`patchedSha256`、`headerBytes`），备份放在安装目录内且不随插件卸载自动删除，直到还原成功并校验。
- 还原必须**双 SHA 校验**：当前文件 == 补丁后 SHA → 写入 → 结果 == 原始 SHA；任一不符立即拒绝并保留备份。
- 应用/还原脚本必须**自带编排**：检测 DSH 进程 → 等待完全退出（进程数为 0 **且** 文件可独占打开，连续多次确认）→ 修改 → 校验 → 重启 DSH。
- 等待方必须是**独立进程**（WMI `Win32_Process.Create` 或计划任务）：DSH host 退出会同步终止子进程树，普通子进程会被一起杀掉。
- 失败时必须**弹窗 + 日志 + 保留备份**，不静默、不重启到坏状态。
- 优先考虑**不碰安装目录**的方案：宿主 service、`desktopNext` 支持的能力、纯 Cordis 插件、外部小进程等。

### 3.4 ⛔ M4 —— 卸载后自检并清理“四件套 + 联接”

卸载脚本结束前必须断言并清理：

```powershell
$profile = "$env:USERPROFILE\.dsh\profiles\desktop"   # 以实际 profile 为准
# 1. package.json: dsh.profile.bundles 无本包、dependencies 无本包
# 2. pnpm-lock.yaml / node_modules\.modules.yaml 无本包
# 3. cordis.yml 无本包的行
# 4. node_modules\<pkg> 联接残留 → 用 rmdir（只删链接，不动真实目录）
cmd /c rmdir "$profile\node_modules\<pkg>"
# 5. 打印最终清单，作为卸载成功的证据
```

建议在 README 给出这段自检命令，用户可自行验证“真的卸载干净了”。

### 3.5 ⛔ M5 —— 用户数据默认保留，清除必须显式

- 数据写入 `~/.dsh/<plugin-scope>/...` 或 `ctx.storageDomain`，不要散落在 `cwd`、桌面或安装目录。
- 卸载**默认保留**用户数据；`--purge` 或单独命令才删除，并在 README 写清会删什么。
- 卸载完成后提示数据位置（保留了什么、怎么删）。

### 3.6 ⛔ M6 —— 卸载脚本的等待 / 超时 / 日志 / 退出码

- 参数：`-TimeoutSeconds`（默认 ≥ 900s，壳补丁类建议 ≥ 3600s）、`-Force`、`-SkipPlugin`、`-DryRun`、`-PurgeData`。
- 全程写日志（含时间戳、pnpm 输出、校验值、错误堆栈），路径在 README 中给出。
- 等待退出时给出明确提示（“请从托盘完全退出 DSH，不要点重启按钮”），并解释“卸载可能静默数分钟是正常现象”。
- 退出码约定：`0` 成功；`1` 失败；`2` 参数错误；`3` 等待超时；`4` 校验失败。README 写明。

### 3.7 ⭐ R1 —— 网络与耗时预期

- 安装/卸载路径尽量**不依赖网络**（本地 `link:` / tarball / 已构建产物）；必须联网时提供离线模式或超时。
- 在 README 提示：DSH 内置 remove 走 pnpm，可能因 registry 重试耗时数分钟；**关闭终端会中断并留下半状态**，请等待完成或查看日志。
- 发现卸载卡住时，先检查 `.plugin-manager/logs/operation-*/pnpm.log`，不要连续重复点击。

### 3.8 ⭐ R2 —— 宿主依赖、版本与包元数据

- `@deepseek-ai/*` 宿主接口包只进 `peerDependencies`（构建用放 `devDependencies`），避免遮蔽宿主（市场 §6.6 真实案例）。
- 声明 `engines.dsh`（如 `">=0.2.0 <0.3.0"`）；版本遵循 semver，每次发版必 bump。
- `main`/`exports`/`files` 指向真实存在的文件；产物型优先（提交 `lib/` 构建产物），源码型提供自包含 `prepare`。
- 填写 `repository`，README 的安装命令必须是**全新 profile 验证过**的命令。

### 3.9 ⭐ R3 —— 运行时资源清理（Host/Client）

- 所有长生命周期资源（route、listener、timer、watcher、DOM、React root、socket、临时 service）必须用 `ctx.effect(() => disposer)` / `ctx.on()` 归属当前 fiber，保证 dispose/HMR 时逆序清理。
- **无 daemon**：DSH 是前台进程，常驻/守护放进程外宿主；子进程一律走 `ctx.subprocess`（host 退出会同步回收子进程树）。
- 有 HMR/dispose 安全测试：卸载/重载后 registry、DOM、style、controller 全部清理。
- 不抢 single 槽（P401），不裸 `child_process.spawn`（P202）。

### 3.10 ⭐ R4 —— 安全与披露

- 按市场 STANDARD §9 声明：`cloud`、`network`、`offline_mode`、`api_keys`（env/storage）、`permissions`、`jurisdiction`、`retention`。
- 权限最小化；凭据只从环境变量读取、不落盘、不进日志；本机 HTTP 端点回环 + 方法白名单。
- 第三方脚本安装必须让用户显式确认；Git 安装建议锁 commit。

### 3.11 ⛔ A —— 反模式清单（不要做）

| 反模式 | 后果 |
|---|---|
| 根目录放 `install.ps1` 又声明 `dsh` 能力 | 市场易判错类型；我应放 `scripts/`（市场 STANDARD §6.1） |
| 插件自己往 profile `cordis.patch.yml` 注册行 | 双加载崩溃（issue #39） |
| 把宿主接口包写进 `dependencies` | 旧副本遮蔽宿主，工具调用全挂 |
| 在 DSH 运行时改 `app.asar` | 文件损坏 / 启动失败 |
| 卸载脚本没有等待退出逻辑 | 中断后半状态，用户以为“卸载烂” |
| 卸载成功但不清 `node_modules` 联接 | 下次安装/更新出现诡异状态 |
| 卸载默认删用户数据 | 不可逆数据丢失 |
| 失败静默 / 只打印英文堆栈 | 用户无法自助恢复 |

---

## 4. 参考实现：幂等卸载脚本骨架（PowerShell）

> 关键点：独立进程等待退出、双 SHA 校验、失败保留现场、自动重启。完整实现可参考 `dsh-restart-button` 的 `uninstall-restart-key.ps1` + 本次修复用的 `restore-on-exit.ps1`。

```powershell
param(
  [string]$InstallDir = 'D:\DSH',
  [int]$TimeoutSeconds = 900,
  [switch]$SkipPlugin, [switch]$Force, [switch]$DryRun, [switch]$PurgeData
)
$ErrorActionPreference = 'Stop'
$profileDir = Join-Path $env:USERPROFILE '.dsh\profiles\desktop'
$backupDir  = Join-Path $InstallDir 'resources\.dsh-myplugin-backup'
$log = Join-Path $env:USERPROFILE ".dsh\myplugin-uninstall.log"

function Log($m) { Add-Content $log "$(Get-Date -Format s)  $m" -Encoding UTF8 }
function Popup($m,$icon) { (New-Object -ComObject WScript.Shell).Popup($m,0,'DSH 插件卸载',$icon) | Out-Null }

# 1) 移除插件包（官方通道；-SkipPlugin 可跳过）
#    dsh plugin --profile desktop remove dsh-my-plugin   （可能耗时数分钟，记录输出）
# 2) 等待 DSH 完全退出：进程为 0 且 app.asar 可独占打开，连续 3 次确认
#    若脚本由 DSH 内启动，必须用 WMI/计划任务以独立进程运行本脚本
# 3) 还原外部副作用（示例：app.asar）
#    node <官方 restore 工具> restore --asar <app.asar>
# 4) 双 SHA 校验：当前 == patchedSha256 → 还原 → 结果 == originalSha256
# 5) 清理残留联接：cmd /c rmdir "$profileDir\node_modules\dsh-my-plugin"
# 6) 清理插件管理器四件套断言（§3.4）
# 7) 用户数据：默认保留；$PurgeData 才删除
# 8) 自动重启 DSH：Start-Process "$InstallDir\DeepSeek Harness.exe"
# 9) 失败：Popup + 保留备份 + 非零退出码
```

**从 DSH 内部触发等待脚本的独立进程方式**（避免随 DSH 退出被杀）：

```powershell
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
  CommandLine = 'powershell.exe -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "C:\path\restore-on-exit.ps1"'
}
```

（注意：脚本文件若含中文，Windows PowerShell 5.1 需 **UTF-8 BOM**，否则解析失败。这是本次踩到的真实坑。）

---

## 5. 我要求自己的 AI 交付前自测的清单

**安装**
- [ ] 全新 profile + 全新 `DSH_HOME` 下，按 README 命令一次安装成功
- [ ] `dsh --profile <p> --dump-config` 能看到本插件层
- [ ] 重复安装无副作用（幂等）

**卸载**
- [ ] 官方 `dsh plugin --profile <p> remove <pkg>` 可移除依赖与层
- [ ] 卸载脚本可重入：连跑 2 次第二次仅输出“已清理”
- [ ] 卸载中断后重跑能收敛（模拟 Ctrl+C）
- [ ] 卸载后 §3.4 四件套 + 联接断言全部通过
- [ ] `-DryRun` 不改任何文件
- [ ] 卸载失败保留备份与日志，可重试

**外部副作用（如有）**
- [ ] 运行中拒绝改 `app.asar`，并给出明确提示
- [ ] 等待退出逻辑用独立进程；超时/校验失败有弹窗与日志
- [ ] 还原后 SHA256 == 原始值
- [ ] 还原后 DSH 可正常启动

**网络 / 兼容 / 安全**
- [ ] 断网环境下卸载仍然成功（或给出明确失败原因）
- [ ] `engines.dsh`、peerDependencies、版本号正确
- [ ] 披露字段齐全；无裸 `child_process`；未抢 single 槽
- [ ] 卸载默认不动用户数据，`--purge` 行为在 README 写清

**验证工具**
- [ ] `npx dsh-plugin-verify <插件路径> --repo <DSH checkout>` 通过（7/7 waterfall + tools/result）
- [ ] 如发布到市场：仓库有 `dsh-plugin` topic、LICENSE、README（安装/卸载/最小示例）、`repository` 字段

---

## 6. 附：对生态的观察（仅供参考）

> 以下仅为我个人对生态的观察，不构成对官方、市场或任何作者的要求。

1. **插件管理器**
   - 支持 `uninstall` 钩子 / 外部副作用声明（§3.2 的 `dsh.install.effects`），由宿主在重启时统一 apply/rollback；
   - 明确“移除中 / 已移除 / 部分完成”状态，`not-removable` 附带诊断与建议（当前重试窗口会误报）；
   - remove 走已解析 lock 的离线快速路径，避免无关包的 registry 重试；
   - pnpm 完成后 reconcile `node_modules`，启动时对账孤儿联接；
   - GUI 流式显示 pnpm 输出与“可能耗时数分钟”的提示。
2. **插件市场 STANDARD**
   - 增加“卸载 / 回滚”章节（本文 §3 可供参考）；
   - 卡片展示 `effects` / `requiresFullExit` / 数据位置，让用户安装前就知道“这个插件会改 app.asar、卸载需要退出 DSH”。
3. **官方文档 publish.zh.md**
   - 增加“卸载与外部副作用”一节：明确 `dsh plugin remove` 的能力边界（只删依赖与层，不负责壳补丁/外部文件）。

---

## 7. 参考来源

**官方**
- 《打包与安装插件》<https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.zh.md>
- 《插件与生命周期》<https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/framework/index.zh.md>
- 官方仓库 <https://github.com/deepseek-ai/deepseek-harness>

**市场 / 验证 / 社区规范**
- DSH-Plugins-Marketplace `STANDARD.md`：<https://github.com/bradeGithub/DSH-Plugins-Marketplace/blob/main/STANDARD.md>
- dsh-plugin-verify（运行时验证 + 评审清单）：<https://github.com/qing3a/dsh-plugin-verify>
- DSH 设计规范整理：<https://github.com/qing3a/dsh-plugin-verify/blob/main/docs/source/dsh-design-conventions.md>
- dsh-plugin-development Skill：<https://github.com/NanmiCoder/dsh-agent-teams/blob/main/.dsh/skills/dsh-plugin-development/SKILL.md>
- dshbase（社区目录/教程）：<https://dshbase.com/>
- awesome-dsh-plugin：<https://github.com/awesome-dsh-plugin/awesome-dsh-plugin>

---

## 附录 A：本次事故时间线（可复现证据）

| 时间 | 事件 |
|---|---|
| 00:02:33 | `dsh-whale-widget` 移除操作启动（pnpm remove） |
| 00:02:35 → 00:06:44 | pnpm 对 `@dsh-ops/bash-win32-x64` 的 registry 请求失败，按 10s/1min 退避重试；`Done in 4m 10.6s` |
| 00:06:5x | 再次调用移除，返回误导性 `not-removable`（bundle 已从列表消失、pnpm 已写完 `package.json`） |
| 00:06:44 后 | `node_modules/dsh-whale-widget` junction 残留（指向桌面 0.3.18 包） |
| 00:15:00 → 00:19:09 | `dsh-restart-button` 移除，同样 `Done in 4m 10.6s`；`node_modules/dsh-restart-button` junction 残留 |
| 00:14 / 00:15 | 作者自带卸载脚本两次卡在“移除插件包”步骤（用户关闭窗口 → 中断） |
| 00:25:03 | 独立后台脚本启动（WMI 逃逸），等待 DSH 完全退出 |
| 00:27:26 → 00:27:30 | 检测到退出 → 官方 restore → 双 SHA 校验通过 → 删除备份 → 自动重启 DSH |
| 00:29 | 复核：`app.asar` 为原始 SHA256、无补丁标记、profile 无残留、Loader 无插件条目 |
---

## AI 与本人调研

本文档由 AI 与本人调研部分文档后生成，不一定全面。
