---
description: "Dshroid 跨端任务控制台的拟议开发计划，包含能力复用、远程访问、任务语义、阶段验收和 GLM 开发交接。"
---

# Dshroid 开发计划

[English](dshroid-plan.md) | 中文

## 摘要

Dshroid 让安卓平板发起、监视和补充 Mac 上的 AI 任务。Mac 持有执行进程、模型凭证和会话日志，两端显示同一份任务数据。第一版交付任务控制闭环、安卓 APK 和 Mac 管理应用；持久化代码 diff 是后续版本。

本文是拟议实现的参考规格，不代表功能已经实现。调研日期为 2026-10-01，代码基线为上游标签 `dsh-v0.2.0-rc.2`，仓库版本为 `0.2.0-rc.2`。开发分支为 fork 中的 `dshroid`；开发期间固定此基线，升级上游另行验证。

## 目录

- [目标与范围](#scope)
- [现有能力与计划修正](#findings)
- [社区代码复用](#community)
- [技术方案](#architecture)
- [任务与提示语义](#semantics)
- [远程访问与设备管理](#remote)
- [开发阶段与工时](#phases)
- [验收与审核证据](#acceptance)
- [GLM 开发交接](#handoff)
- [开发备注](#dev-note)

-----

<a id="scope"></a>
## 目标与范围

用户在平板选择 Mac 已登记的项目，提交目标和约束，查看实际执行过程，补充提示，回答问题或审批操作，并看到执行结束原因。平板断网和应用退到后台不应取消 Mac 上的任务。

| 第一版交付 | 明确推迟 |
| --- | --- |
| 单用户、单 Mac；同一项目最多一个 Dshroid 执行任务 | 多机调度、团队权限、定时任务、插件市场 |
| 项目选择、任务列表、详情、发起任务 | 平板代码编辑、完整 IDE、远程桌面 |
| 中途提示、后续消息、停止、问题与审批 | 自动合并、自动提交或推送 Git 变更 |
| 私有网络中的 HTTPS 访问与可撤销设备配对 | 公网匿名接入、作者运营的中继、远程唤醒 |
| Mac 后台服务、管理窗口、安卓 APK | 系统推送、应用商店发布与完整安装器 |
| 结果摘要、已记录的工具及测试结果 | 重启后仍可审阅的持久化 diff |

第一版允许 AI 按授权执行代码修改，但不自动制造“只读分析模式”：若以后提供该模式，必须用工具能力和执行权限落实，不能只靠提示词。项目在 Mac 本地登记；平板不能提交任意本机路径或改变模型密钥。

开发验收使用指定的真实 Mac 和安卓平板。阶段 0 记录 macOS、Android、WebView、浏览器、Node 和 Tailscale 的实际版本；Android 最低版本由真机与构建工具验证后写入配置。首版发布物用于个人安装，正式签名、公证和商店分发另行估算。

-----

<a id="findings"></a>
## 现有能力与计划修正

先复用官方机制，再增加产品需要的薄层。下表来自当前 checkout 的源码和 README 静态检查；真实模型、网络和设备行为仍须通过阶段 0 验证。

| 能力 | 现有依据 | 本项目处理 |
| --- | --- | --- |
| 创建、历史、提示、队列、取消 | [Session Controller](../packages/api/session-controller/README.zh.md) | 复用控制器和生成的 Remote 类型 |
| 请求去重 | [prompt 实现](../packages/api/session-controller/src/commands.ts) 的 `requestId` / `rpcId` | 同一次提交的网络重试保留身份 |
| 实时连接 | [Gateway](../packages/api/gateway/README.zh.md) 的 `/api/remote.mux` | 使用现有 WebSocket 多路复用，不另造 SSE 协议 |
| 重连补齐 | Gateway 的 journal/snapshot streams 与 Session Client | 复用序列、分页和连接代次处理 |
| 项目与导航 | [Workspace Controller](../packages/api/workspace-controller/README.zh.md) | 项目登记在 Mac，平板读取允许列表 |
| 用户问题与审批 | [User Questions](../packages/interaction/user-questions/README.zh.md)、[User Approval](../packages/interaction/user-approval/README.zh.md) | 复用现有请求、答案和撤销语义 |
| 会话与进程生命周期 | [Agent](../packages/core/agent/README.zh.md)、[Agent Loop](../packages/core/agent-loop/README.zh.md) | 运行权属于 Host，与客户端保留引用区分 |
| 当前轮次 diff | [Workspace Changes](../packages/deliverables/workspace-changes/README.zh.md)、[UI Deliverables](../packages/client/ui-deliverables/README.zh.md) | 可以借用展示组件；不能承诺跨重启历史 |
| SDK | [TypeScript SDK 限制](../packages/sdk/client/README.zh.md#known-limitations-and-deferred-work) | 首版不以 SDK 子进程协议承载平板控制 |

原草案方向合理，但应取消独立的 AI 执行引擎、第二份聊天事件流和完整任务调度数据库。MVP 中一条任务就是一条 Dshroid 管理的根 Session；轮次是执行记录，提示是 Inbox 消息。只持久化无法从现有 Session 重建的产品元数据。

原草案的“下一次模型调用采用提示”必须以实际入队与请求记录区分；“成功”必须与目标验证分开；“取消”必须与清空后续工作分开；“Mac 常驻”必须由独立服务生命周期实现。原来的 100–142 小时不足以稳妥覆盖自有认证、两端应用和异常恢复。

官方 [Connection](../packages/client/connection/README.zh.md) 使用启动 token 换取 authority 绑定的 cookie，并检查 Host 与 Origin。当前 cookie 不带 `Secure`，没有逐设备退出机制；Host 信任名单也不是认证。不能通过开放监听地址或放宽信任名单就宣称远程方案完成。

-----

<a id="community"></a>
## 社区代码复用

复用决策必须同时满足许可证、当前接口、所需权限和真机链路四项条件。以下是 2026-10-01 的静态检查结果，未运行这些项目的测试，也未安装插件；版本声明不等于兼容性证据。

| 候选 | 已检查事实 | 决策 |
| --- | --- | --- |
| [dsh-web](https://github.com/zhu1090093659/dsh-web/tree/dev/packages/dsh-remote-web-ui) | remote-web-ui 为 `0.4.4`，声明 DSH `>=0.2.0-rc.2`，Apache-2.0；源码包含 mux 转发、设备配对和撤销 | 配对与重连实现优先参考；不整包默认安装 |
| [dsh-mobile-gateway](https://github.com/agent-mobile/dsh-mobile-gateway) | `0.4.0`，MIT；含白名单与 WebSocket，但其 mux adapter 使用旧 `wireStream.open` 参数方式，和本基线不一致 | 参考限制路由与认证测试，迁移后才能采用 |
| [Smart-DSH](https://github.com/hikarioyama/Smart-DSH) | `dsh-notify-push` 模块声明 MIT，根仓库未检出统一许可证；模块面向 Android Chrome Web Push | 后续通知参考；复制前核实具体文件授权 |
| [dsh-telegram](https://github.com/Kevin66Z0/dsh-telegram) | MIT；远程消息渠道，产品交互与本项目不同 | 不进入 MVP 依赖 |
| [dsh-lark](https://github.com/sugarforever/dsh-lark) | MIT；Lark 消息桥接 | 后续通知渠道参考 |

`dsh-web` 的配对通道设计允许较全面的 Host 控制，并包含可选作者中继与遥测代码。Dshroid 需要任务级访问限制，因此不能把“扫码可用”当作授权模型合格，也不能直接继承这些网络服务。此前把这些插件概括为仅依赖 SSE 或全部旧接口不准确，应分别审查 HTTP、mux、回答回传和重连。

阶段 0 为每个实际采用的组件记录版本、许可证文件、调用的官方入口、所需修改和验证结果。依赖采用锁文件中的精确解析；移植源码保留版权、许可证及适用的 NOTICE，并记录来源文件和本地修改。优先引用维护中的依赖，避免复制整个社区仓库及其旧协议实现。

-----

<a id="architecture"></a>
## 技术方案

React 任务界面复用官方 Client 的连接、会话和投影能力。Mac 的唯一 Node Host 运行模型、工具和持久化；原生应用只管理服务或显示界面。所有支持的 Node 应用仍通过 [dsh profiles](architecture.zh.md#application-launch) 启动。

| 部件 | 拟议职责 | 不承担的职责 |
| --- | --- | --- |
| Dshroid profile / bundle | 在官方 Web/base 组合上挂载必要插件，固定依赖，关闭不需要的功能 | 另建 CLI bin 或绕过启动器 |
| Dshroid Host 插件 | 受限任务控制、设备授权、停止意图、最少元数据 | 复制 Agent Loop、Session 日志或队列 |
| Dshroid Client 插件 | 任务列表、详情、提示回执、等待用户状态 | 根据前端计时器推断执行结果 |
| macOS `launchd` LaunchAgent | 用户登录期间托管唯一 Host；管理启动、退出和崩溃重启 | 保证注销、关机或睡眠时继续执行 |
| macOS 管理应用 | SwiftUI 菜单栏、服务状态、打开任务页、配对管理；需要内嵌时用 WKWebView | 自己再启动第二个 Host |
| Android 应用 | Kotlin WebView 显示同源 HTTPS 任务页，处理返回、前后台和连接错误 | 在平板运行 Node 或模型工具 |
| Tailscale Serve | 私有 tailnet 中的 HTTPS 访问和到 loopback 服务的代理 | 替代 Dshroid 的设备授权 |

默认采用原生平台的薄壳，避免为显示同一个 Web 页面同时维护 Electron 与 Tauri 两套桌面运行时。官方 Electron 仍是独立产品，它目前拥有自己的 Host；不能原样套用并同时启动 LaunchAgent。若阶段 0 证明现有 Electron 能以小范围改动连接外部 Host，可以以明确验证证据替换 Mac 薄壳方案。

Android WebView 只载入已配对的 HTTPS origin，外部链接交给系统浏览器，不开放任意 JS 原生桥，不忽略 TLS 错误，不加载本地文件或混合 HTTP 内容。使用 [Android WebView 官方接口](https://developer.android.com/develop/ui/views/layout/webapps/webview)；Mac 内嵌页面使用 [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview)。业务界面只有一份 React 实现。

Node 使用仓库支持的版本范围。Mac 服务使用安装包中明确的 Node 和 dsh 路径，不依赖 LaunchAgent 的 shell PATH；源码开发通过现有 source launcher。使用单独的 Dshroid Harness home，与用户已有的官方配置和 Session 数据隔离；密钥由 Mac 本地配置，不复制用户原数据。

拟议源码位置遵守现有分组：Host/API 特性放入 `packages/api/` 的新功能包，UI 放入 `packages/client/`，组合放入 `packages/bundle/`，原生壳放入独立 `apps/dshroid-*` 目录。阶段 0 确定最小包数、实际名称和 profile 安装流程；这些路径不是已存在的接口。

新增包按 [package cookbook](cookbook/adding-a-package.zh.md) 登记，并明确 Host / Client 编译配置。部署相关端口、公开 origin、配对有效期、设备上限、请求限额和可选防睡眠配置由验证过的 Config 管理。工具和审批使用官方沙箱及权限策略。

-----

<a id="semantics"></a>
## 任务与提示语义

执行状态、连接状态和结果验证分别表示，避免断网、等待问题和轮次结束产生误导。状态由 Host 事实与持久日志重建，而不是维护一份独立前端状态机数据库。

### 数据与身份

使用现有 branded `SessionId`、`WorkspaceId`、`MessageId` 和 `SessionRequestId`。最少产品元数据包括 Dshroid Session 归属、创建请求身份、项目关联、停止意图和可选 Git 基准事实；非会话数据由现有 [Storage](../packages/storage/README.zh.md) 能力保存，设备凭证由 [Credentials](../packages/credentials/README.zh.md) 能力管理。日志、提示队列和结果文本保留官方唯一数据源。

创建操作先以稳定请求身份登记意图，为同一次创建绑定稳定 Session 身份，再创建或恢复 Session、发送初始提示。进程在任一步骤退出后，只补齐缺失的登记或入队步骤；已接收的 prompt 不重复发送为新身份。不同内容复用同一身份返回冲突，不静默返回另一个任务；网络响应丢失显示“正在确认提交结果”，不新建请求。

每个项目的 Dshroid 执行 admission 串行化；MVP 不建立任意长度的自动调度队列。项目忙时拒绝另一条任务执行并保留草稿；当前任务内部仍使用官方 Inbox。停止中拒绝新提示，停止完成后只有显式继续才能再次唤醒。

### 提示与停止

| 操作 | 官方语义或拟议实现 | 用户可见回执 |
| --- | --- | --- |
| 发起 / 后续消息 | `prompt(mode: 'queue')` → `followup`，进入 `next-turn` | 已接收、排队、进入执行 |
| 补充当前任务 | `prompt(mode: 'steer')` → `steer`，进入 `next-step`；idle 时唤醒 | 已接收、等待下一步、已进入请求 |
| 后续消息编辑 / 删除 | 官方 queue mutation，已 claim 的消息拒绝修改 | 明确显示成功或已进入执行 |
| 停止当前轮次 | 官方 `session.cancel`，保留 Inbox | 标注“后续消息仍可能继续”，不作为默认停止按钮 |
| 停止任务 | Dshroid Host 串行登记停止意图，调用默认清空 Inbox 的 `agent.cancel`，停止该任务拥有的活动并等待收敛 | 停止中；实际收敛后已停止 |

当前 [运行时实现](../packages/core/agent-loop/src/agent.ts) 中 `steer` 只调用 `send(input, 'next-step', true)`，不主动 abort 当前模型请求或工具。它可能错过已经 claim 输入的请求，也可能在取消之后被归到下一轮。Client 注释中的“interrupts”不能作为中途强制停止的保证；阶段 0 应固定实际行为的集成证据。

“已进入请求”只在该消息被记录并包含在实际模型请求输入后显示，不能单凭 HTTP 成功、Inbox claim 或 Assistant 新输出判定。请求发出不代表模型遵从了约束；消息在停止时被清除则显示已移除。若控制器现有投影不足，应增加由现有事件推导的投影，不另造假定成功的回执。

停止不撤销已经发生的文件写入或外部操作。停止意图必须阻止晚到的 Dshroid 提示重试唤醒任务；任务拥有的子代理和后台 jobs 用各自 owner API 取消并等待，不杀掉其他 Session 的进程。MVP profile 不挂载自动调度和自动恢复目标。无法终止的操作保持“停止受阻”及活动详情，不能超时后伪报已停止。

### 状态与恢复

| 维度 | 值及依据 |
| --- | --- |
| 连接 | 连接中、在线、离线；离线同时显示最后更新时间，任务状态标为最近观测 |
| 执行 | 排队、执行中、等待审批、需要回复、停止中、已停止、已结束、失败、受阻、执行中断 |
| 结束原因 | 从 `turn/end.reason` 投影；`completed` 是执行结束，`error` / `aborted` / `interrupted` 分别呈现；token 上限和未知扩展原因不能误报完成 |
| 目标验证 | 未验证、已有测试证据、存在未解决问题；首版不做自动“成功”评分 |

当前轮次正在等待审批时显示等待审批；问题已转为 continued 且执行继续时显示“需要回复”提示，不伪装成阻塞等待。展示实际工具操作和日志，不凭消息措辞编造“定位→修改→测试”阶段或百分比。由模型自述的测试与已记录命令、退出状态分开标识。

客户端重连读取控制 baseline 并补齐持久 journal；流式 token 是临时展示，不能作为唯一历史。复用官方流订阅、分页、重复消除和旧代次拒绝；浏览器锁屏后允许连接关闭，恢复前台重新连接。

Host 崩溃或重新启动后保留历史，按官方机制把未闭合轮次识别为 `interrupted`，界面显示需要用户决定是否继续。登录服务可以重启，但不能因此自动重做 shell、网络写入或旧审批。读取冷 Session、重复确认创建等操作不得意外触发执行；恢复 Inbox 的 admission 必须受到停止意图和显式继续控制。

### 代码变更与后续 diff

首版同一项目只有一个 Dshroid 写任务，但这不能隔离用户在 Mac 或其他工具中的修改。任务开始时记录 Git HEAD、工作目录和起始 dirty 状态，只表示基准事实；不宣称可从 HEAD diff 准确归属任务，非 Git 项目明确显示没有 Git 基准。

后续引入 task-owned worktree 与持久内容快照后再交付 diff。为 dirty 源目录明确选择“只使用已提交基准”或“显式保存起始未提交内容”，不偷偷 reset、stash 或复制。覆盖 tracked/untracked、删除、重命名、二进制、超大文件及重启恢复；只保存 commit 或 path 不足以重建修改前后的内容。worktree 是代码隔离，不是进程沙箱。

-----

<a id="remote"></a>
## 远程访问与设备管理

默认用 [Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve) 在私有 tailnet 中提供 HTTPS，Mac Node 服务监听 loopback。两端安装并登录同一受限 tailnet；阶段 0 验证所选 Mac Tailscale 发行版支持端口代理、mux upgrade 和长连接。公网 Funnel、自动隧道和第三方稳定域名不纳入默认方案。

Dshroid 在同源任务入口提供设备会话，先验证设备，再检查操作和项目归属。官方 owner cookie 与配对设备凭证不能互相代用。Mac 本地管理入口需要官方本机认证；不能只凭来源 IP 为 loopback 就信任 Tailscale 代理后的请求。

配对由 Mac 生成短期、一次性随机凭证，通过二维码或手工粘贴接受，并在 Mac 确认设备后换成可撤销设备会话。MVP 不要求相机扫描：平板复制配对链接即可。HTTPS cookie 使用 `HttpOnly`、`Secure` 和合适的 SameSite / Path；页面不把长期凭证放入 URL 或 localStorage，日志不记录凭证。

精确校验配置的 public origin、HTTP Host、浏览器 Origin、WebSocket upgrade Origin 和请求尺寸；信任代理头只限指定 loopback 代理，并验证部署配置。未经认证的页面 bootstrap 不包含工作区、路径、配置或密钥。UI、HTTP 与 WebSocket 尽量同源，避免跨域 cookie 和 WebView 第三方 cookie 依赖。

| 允许的远程操作 | 必须在 Host 拒绝的操作 |
| --- | --- |
| 读取允许项目和 Dshroid 根 Session 列表、历史、状态 | 任意目录访问、无归属 Session 或项目 |
| 创建任务、提示、队列修改、停止、显式继续 | 插件安装、配置修改、密钥读取、升级与电源控制 |
| 回答本任务的有效问题与具体审批 | 任意 shell/terminal RPC、任意文件下载和伪造审批身份 |
| 读取已授权任务的必要结果 | 不受限的 `$events`、任意 scoped receiver 与事件 reply |

任务本身仍能按工具权限执行 shell；禁止的是平板绕过任务与审批直接调用任意 shell API。调用过滤必须覆盖 HTTP unary、同一 mux 的每次 logical open、uplink 和回答回传，不只过滤 URL。持久 journal 只按已授权 Session 读取；实时事件不能广播其他 Session 或全局配置。

优先使用官方 Gateway 的协议、生成类型和公开 dispatch 能力，补充最小设备 admission 与过滤扩展；不复制协议服务端，也不直接访问私有成员。当前官方连接把所有调用视为 operator，逐设备和逐任务授权是新增工作。阶段 0 必须验证公开扩展是否足够；不足时提出最小 typed admission 扩展及测试后实现，不能退回到整条 `/api` 无条件代理。

撤销设备同时拒绝新请求、关闭它的已建立 mux 和等待用户处理权，并清理对应 cookie / native 配对记录；不取消已经在 Mac 执行的任务。审批绑定请求、Session、具体命令或文件、工作目录和有效期，两端同时操作只能形成一份决定。审批在停止、工具取消或 Host 重启后过期，不自动重放旧答案。

-----

<a id="phases"></a>
## 开发阶段与工时

按可独立审核的交付分阶段推进。以下是开发与验证工时的估算，不是 GLM 的固定完成承诺；阶段 0 后依据实测重新估算。真实模型调用成本和用户真机验收时间另计。

| 阶段 | 工作与交付 | 放行标准 | 估算 |
| --- | --- | --- | --- |
| 0：能力验证 | 固定官方基线，建立真实 Mac→安卓浏览器控制实验；核实 prompt、steer、cancel、问题、审批、重连和社区候选 | 有实际链路证据、许可清单、profile 草案与远程 admission 方案；明确不能复用之处 | 12–18 小时 |
| 1：任务控制 | Session 归属、创建去重、项目串行 admission、提示回执、停止意图、恢复策略 | 可重复提交且不重复执行；停止清空所属工作；崩溃不自动重做操作 | 24–36 小时 |
| 2：受限远程 | Tailscale HTTPS、配对、设备撤销、HTTP/mux 授权、问题与审批桥接 | 未授权读写全部拒绝；撤销已连设备立即失权；跨网络闭环可用 | 28–42 小时 |
| 3：平板界面 | 列表、详情、发起三页；需要回复提示、草稿、回执、停止与错误恢复；复用官方投影和 UI primitives | 横竖屏、中文输入、软键盘、旋转、锁屏恢复通过；状态不误导 | 24–36 小时 |
| 4：两端应用 | LaunchAgent、Mac 菜单栏管理、固定 Node/runtime 路径；Android WebView APK 与生命周期 | 关窗口或退出管理壳不结束 Host；退出服务需要明确动作；APK 真机可安装 | 24–36 小时 |
| 5：稳定性与交接 | 端到端故障场景、相关 snapshots、录屏、运行与卸载说明、审核材料 | 下节验收全部通过；没有隐藏权限升级或数据破坏 | 20–28 小时 |
| 后续版本 | task-owned worktree、持久 diff、平板审阅；通知另列独立任务 | 重启后内容可验证，源目录已有修改不混入 | 28–44 小时 |

MVP 合计 132–196 小时，加约 20% 集成余量约 158–235 小时。每周投入 15 小时约为 11–16 周。第一批可用浏览器闭环在阶段 0–3 内交付；不要等 APK 和 Mac 包装完成才验证核心体验。

阶段 0 中，真实模型检查只有在具备用户已有 API 配置时运行；无凭证则明确标记未验证并提供可执行的人工步骤。先用 keyless fixture 覆盖控制协议，再由真机补足网络和生命周期事实。未通过远程授权验证前，不进行公网部署。

每阶段单独提交可 review 的改动，依赖阶段完成再继续。UI mock、keyless 集成和真机验证都保留证据，不能用“静态检查通过”替代用户流程。前端不先编造 backend 已有的接口；阶段 0 / 1 明确 generated Remote 入口后再接 UI。

-----

<a id="acceptance"></a>
## 验收与审核证据

以下场景应通过官方入口或实际 Dshroid profile 执行，避免测试直接挂载插件而遗漏启动、权限或网络问题。交付记录每个命令、环境和结果，失败或未验证项必须保留。

| 场景 | 必须观察到的结果 |
| --- | --- |
| 创建响应丢失、连点提交、Host 在创建中退出 | 一个逻辑请求对应一个 Session 和一条初始提示；身份冲突明确拒绝 |
| 模型请求中、工具执行中和 idle 时补充提示 | 当前执行按实际语义完成；提示进入最近可接纳请求；没有虚假的已生效回执 |
| 队列中已有消息时停止 | 取消真实活动、清空所属 Inbox；晚到重试不再次唤醒；其他任务不受影响 |
| 子代理、长命令或 job 尚未结束 | 只有 owner API 确认收敛后显示已停止；无法取消的活动仍可见 |
| 平板切网、锁屏、后台、应用被杀、旋转 | Mac 执行继续；恢复后状态与持久日志无重复、无缺口；草稿按策略保留 |
| Mac 窗口关闭、管理壳退出、Host 崩溃、睡眠 | 前两者不结束服务；后两者显示失联/中断；恢复不重放有副作用操作 |
| 无凭证、撤销设备、过期配对、错 Origin | HTTP 与 mux 都拒绝；已连设备失权；凭证不出现在日志或截图 |
| 越权 Session、伪造 path、直接调用全局接口或 `$events` | Host 拒绝，且响应不泄露其他项目的数据 |
| 两端同时答题/审批，或停止后发送旧答案 | 只有有效决定被接受；过期身份不生效 |
| 轮次结束、模型错误、token 上限和未运行测试 | 显示实际结束原因；不因 `idle` 或 `completed` 宣称目标成功 |

非平凡的模型/产品可见变化更新 keyless recorded-session snapshots，遵守 [snapshot ownership](../snapshots/AGENTS.md)。界面几何和 ARIA 预期放 owner-local，UI 变更提供浏览器 GIF。真机证据记录 Android WebView 和 Mac 生命周期；截图不替代断线与停止的行为验证。

每个阶段根据 [pre-push checks](../.agents/skills/dsh-pre-push-checks/SKILL.md) 选择相关命令：行为运行 focused tests，可见输出运行 owning snapshots，Remote 改动做 typecheck，包与构建入口改动做 build/hygiene/built smoke，文档做 doc-sync。全量覆盖与平台矩阵由 CI 执行，不默认在本地重复完整测试集。

遵守根 [AGENTS.md](../AGENTS.md)、[架构](architecture.zh.md)、[生命周期规则](defensive-patterns.zh.md) 和 [测试政策](testing.zh.md)。新增模型输入必须可由 Session log 重建；声明的持久类型变化提交 [type acknowledgement](cookbook/reviewing-persistence-type-changes.zh.md)，已发布 Session generations 不修改。对用户可感知的破坏性变化立即写 upgrade guide；保留既有工具沙箱与审批机制。

Mac 日志轮转并脱敏；防睡眠仅在用户选择且任务活跃时持有 assertion，活动结束或插件 dispose 时释放，不承诺阻止合盖和远程唤醒。停止服务和卸载必须明确显示活动任务，移除服务注册与设备访问，保留 Session 数据直到用户另行选择删除。

交给审核者的材料包括：阶段改动说明、实际命令与结果、snapshot/GIF、真实设备记录、依赖和许可来源、已知限制、安装/升级/停服/卸载方法。签名密钥、API key、配对凭证和完整个人日志不进入提交。

-----

<a id="handoff"></a>
## GLM 开发交接

将下面的指令连同本文交给 GLM。本文提供产品与验收方向，现有类型、生成描述和源码负责实际接口；实现发现冲突时先修正规格，再提交行为与验证证据。

> 在 fork 的 `dshroid` 分支开发 Dshroid，先阅读 `AGENTS.md`、`docs/architecture.md` 和 `docs/dshroid-plan.zh.md`。先完成阶段 0 并提交验证记录，再按阶段 1–5 实现。复用官方 Session、Inbox、Gateway、投影、Workspace、问题和审批；不要另建 Agent Loop、完整调度引擎或事件传输协议。所有 Node 服务经 dsh profile 启动。真实远程设备必须通过受限 admission，不能整条代理 `/api` 或仅隐藏配置按钮。每阶段提交最小可审核改动，运行与范围匹配的测试，记录真实结果和未验证项；UI 变化交付 snapshot 和 GIF。不要自动提交/推送用户项目的代码，不自动合并开发分支。实现差异同步修改本文及英文对应文件。完成后提供提交记录、两端安装物、运行文档与审核证据，由 Codex 审核。

阶段 0 的交付记录至少包含实际版本、真实 profile 组合、API/事件映射、提示接收与请求包含的证据、停止与队列行为、授权入口设计、采用组件的许可证和复用决定，以及阶段 1 的具体源码位置。没有实际网络或 API 凭证时先完成 keyless 工作并列明真机缺口，不能声称阶段 0 已完全验收。

后续 Codex 审核以本计划的可观察结果为准，重点检查重复执行、停止收敛、跨设备授权、异常恢复、日志重建和发布物启动方式。审核发现阻塞问题时修复并复验，再进入个人使用发布。

-----

<a id="dev-note"></a>
## 开发备注

本次仅完成源码/文档静态核实与计划编写，没有开发 Dshroid、安装社区插件、运行真实模型或进行安卓真机联调。阶段工时、薄壳选型和远程 admission 的具体扩展位置是待阶段 0 验证的工作假设。正文中的拟议行为和阶段验收是开发要求，不是当前产品能力声明。
