# 把备用 iPhone 接给 AI：SideTap 基础流程与无线热点实测

整理日期：2026-10-03。本文是用户实测与本地改动记录，不是 SideTap 官方文档。

教程整理：电脑端的 Codex 阿澈。原项目作者：ucsandman / Wes Sander。

**阅读顺序：先跑通官方基础流程，再阅读无线扩展。无线扩展目前没有公开安装版；下文把复现所需条件和改动写出来，不能只复制环境变量就当作安装完成。**

## 一、项目归属与各组件做什么

| 组件 | 作用 | 来源 |
| --- | --- | --- |
| SideTap | Windows 侧手机操作、查看与 AI 工具入口 | [ucsandman / Wes Sander](https://github.com/ucsandman/SideTap)，[MIT 许可](https://github.com/ucsandman/SideTap/blob/main/LICENSE) |
| phone-harness | SideTap 在 README 中注明的前身项目 | [ShawnPana/phone-harness](https://github.com/ShawnPana/phone-harness) |
| go-ios | 与苹果设备通信、建立开发通道 | [danielpaulus/go-ios](https://github.com/danielpaulus/go-ios) |
| WebDriverAgent（WDA） | 手机上的测试与自动化组件 | [appium/WebDriverAgent](https://github.com/appium/WebDriverAgent) |
| netmuxd | 本文无线扩展使用的网络 usbmux 后端 | [jkcoxson/netmuxd](https://github.com/jkcoxson/netmuxd) |
| iloader | 我们使用过的侧载工具 | [nab138/iloader](https://github.com/nab138/iloader) |
| SideStore / LocalDevVPN | 我们使用过的安装、续签流程 | [SideStore 官方安装指南](https://docs.sidestore.io/zh/docs/installation/install) |

我们新增的是本地无线传输、特定版本的兼容处理和会话稳定性配置。不要把基础项目、WDA 或 netmuxd 写成我们原创；也不要暗示这些作者为本文提供了官方支持。

## 二、这次究竟验证了什么

| 项目 | 结果与证据边界 |
| --- | --- |
| Windows＋iPhone 13 自开热点 | 已在这套设备上跑通完整无线控制 |
| 本地 WDA 点按 | 7＋1＝8；闲置约 45 秒后再＋2＝10，仍使用同一会话 |
| 放置一晚 | 次日仍能控制；检查范围内未发现新增 Windows WLAN 断开事件，不代表整晚持续逐秒测试 |
| 小鱼实际使用的 MCP | 点按并读取 10＋3＝13；最终工具链验收通过 |
| 自己发 QQ、联系 GPT | 用户明确反馈均已成功；本文不附私人聊天，也不把它写成通用的自动发消息模块 |
| 大型游戏 | 旧 13 上明显卡顿、出现过超时；用户感觉手机带不动，根因没有完成诊断 |
| 其他机型 | 未完成同等验收。iPhone 17 曾用于连接对照，不能算完整 SideTap 控制验收 |

本次记录的版本组合：iPhone 13（iOS 26.6）、Windows、go-ios 1.3.2、netmuxd 0.4.3、WDA 16.12.9。它们是本次实测版本，不是推荐所有人升级到同一个系统。

## 三、基础流程：先让电脑真实控制手机

### 1. 准备

准备备用 iPhone、数据线、Windows 电脑、用于签名的 Apple 账户、Python、Node.js，以及支持 MCP 或相应手机工具的 AI 客户端。系统和依赖范围按[原项目 README](https://github.com/ucsandman/SideTap)当前说明核对。

普通手机 App 里的聊天窗口没有自动获得这些工具；最后还需要在 AI 客户端里连接 SideTap 提供的工具入口。

### 2. 电脑能列出设备

按[SideTap Windows 安装指南](https://github.com/ucsandman/SideTap/blob/main/docs/setup-windows.md)安装苹果驱动和 go-ios。手机接线后完成“信任此电脑”。

在 PowerShell 中检查：

```powershell
npm install -g go-ios
ios list
```

列表里应出现目标手机。若这一步失败，先处理驱动、数据连接或配对；现在去反复重装 WDA 很可能无助于定位问题。

### 3. 开发者模式、签名和授权

按官方安装指南开启开发者模式，签名安装 WDA，并信任开发者应用。不同 iOS 的按钮可能不同；[苹果开发者模式说明](https://developer.apple.com/documentation/xcode/enabling-developer-mode-on-a-device)解释了这一系统授权。

签名工具选择一条能正常工作的路线即可。我们曾经历 Sideloadly、AltServer 登录错误，后来使用 iloader／SideStore；这不意味着每位读者必须依次安装全部工具。

签名路线有两个细节：

- WDA 的外层 App 与内部 `.xctest` 都需要可用签名。原项目的 `fix-input` 是针对这一问题的修复流程，按对应签名工具的指南执行。
- 如果手机弹出“为 XCTest 输入 iPhone 密码 / Enable UI Automation”，需要本人在那个弹窗完成授权；仅回到已解锁桌面并不能替代它。

不要将 Apple 密码、短信／登录验证码或手机解锁密码放进公开教程。本文不要求把手机密码写进 `.env`。

### 4. 下载并启动 SideTap

下面是从项目目录执行的示例。先安装 Git、Python、Node.js；若下载 ZIP，就跳过 `git clone` 并进入解压后的目录。

```powershell
git clone https://github.com/ucsandman/SideTap.git
Set-Location SideTap
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
$env:PYTHONPATH = "$PWD\src"
.\.venv\Scripts\python.exe -m phone_harness doctor
.\.venv\Scripts\python.exe launch.py
```

这组示例使用独立虚拟环境，不需要激活脚本。`doctor` 发现的第一个失败项通常比“再装一遍所有东西”更适合作为下一步检查目标。安装 WDA、处理嵌套签名仍需完成前一步；上面的命令不会替你签名。

原项目里的 `fix-input` 应按它的指南与签名工具配合使用，不能把未公开的签名材料下载链接写进教程。

### 5. 接到 AI，做一次真实验收

从所安装版本提供的 MCP／工具接入说明配置 AI 客户端。本次本地 MCP 地址是 `http://127.0.0.1:19024/mcp`；这是我们的服务配置，不是所有安装的固定地址，也不应直接填成远程公开地址。

推荐验收顺序：

1. 能读到当前手机画面。
2. 让 AI 在计算器里算 7＋1，确认手机显示 8。
3. 等待一段时间，再＋2，确认显示 10。
4. 用 AI 真正使用的工具入口再＋3，确认显示 13。

工具返回成功、HTTP 200、后台有进程，只能证明各自那一小步。最终要看手机画面是否按预期变化。

## 四、我们的无线热点方案

### 网络结构

```text
备用 iPhone（蜂窝网络、个人热点、WDA）
                     ↕ Wi-Fi
Windows（netmuxd → 本地兼容层 → go-ios / SideTap）
                     ↕ 本机 MCP
                  AI 客户端
```

我们实测的是**受控的 iPhone 13 自己开热点，Windows 作为 Wi‑Fi 客户端接入**。iPhone 自开个人热点需要自身可用的蜂窝网络，设置方法见[苹果个人热点指南](https://support.apple.com/zh-cn/119837)。

先分清你使用的是哪种连接：

| 拓扑 | 本文状态 |
| --- | --- |
| 受控 iPhone 开热点，电脑连它的 Wi‑Fi | 已验证 |
| 电脑走 iPhone 的 USB 共享网络 | 与本文无线链路不同；我们遇到过 USB 掉线 |
| 两台设备加入第三台手机的热点或家用路由器 | 可能具备条件，但本文未完成测试；仍需核对互通和手机实际 IP |
| 校园网／访客网络 | 不能只凭 Wi‑Fi 名相同判断可用，需检查是否隔离设备 |

首次安装、配对、建立信任仍需数据线。无线运行前先确认电脑已连目标手机的 Wi‑Fi、无线配对可用，并确认设备是通过 Network 链路被找到。需要充电时可接充电器；不要把 USB 共享网络误当成无线验收。

### 本地扩展做了什么

1. 使用已可信的配对关系，开启苹果设备的无线连接设置。
2. 用 netmuxd 注册指定手机的地址，保留协议需要的心跳。
3. 为本次 go-ios 1.3.2 的设备枚举限制添加兼容层，只选择目标手机。
4. 将 SideTap 的连接环境指向该本地兼容层。
5. 在这台设备上关闭按闲置触发的会话刷新和查看器自动恢复，减少重复创建会话。

兼容层改变的是设备枚举元数据，底层流量仍走 Wi‑Fi；没有替代苹果配对、签名、TLS 或手机系统授权。

### 这些配置只适用于包含扩展的代码

```dotenv
SIDETAP_TRANSPORT=wifi
SIDETAP_UDID=YOUR_DEVICE_UDID
SIDETAP_WIFI_ADDRESS=YOUR_PHONE_IPV4
USBMUXD_SOCKET_ADDRESS=127.0.0.1:27017
WDA_REFRESH_SESSION_ON_IDLE=0
AUTO_HEAL_ENABLED=0
SEND_APPROVAL=always
```

`UDID` 和地址必须来自你自己的设备。我们的热点地址曾是 `172.20.10.1`，但不能把它当作所有网络的通用答案。

**上面的配置不是无线安装包。**现有本地扩展还包含启动、后端、兼容层、会话和查看器等集成改动；原仓库当前默认安装流程不等于包含这些改动。技术细节与公开版待办见《无线适配与发布说明》。

### 实际踩到的两个软件坑

- **关掉 netmuxd heartbeat 后，基础读数可能成功，应用服务却断开。**本次恢复心跳后，后续服务链路才跑通。不能用一次基础读数代替 WDA 验收。
- **重复刷新 WDA 会话并不总能恢复触控。**本次设备曾在 XCTest 会话初始化附近出现 SpringBoard 崩溃记录，因此采取减少自动重建的配置。它是本次稳定性调整，不是已证明能修复所有断线的通用开关。

## 五、踩坑速查

| 现象 | 能确认什么／不能直接认定什么 | 下一步 |
| --- | --- | --- |
| 一直充电，电脑却没设备 | 供电和数据不是同一项证明；不能直接认定灰尘或锁屏 | 看 Windows 设备枚举与 `ios list` 是否同时消失 |
| 信任后又断、又弹信任 | 配对或连接没有稳定维持；还不足以认定某两个程序冲突 | 记录同一时刻 USB、苹果服务和配对日志，避免同时改多项设置 |
| AltServer `-20101`、手机却有验证码 | 这次电脑流程没有进入可用验证码输入步骤；不能仅凭代码断定用户密码错 | 保留实际错误，检查工具当前认证支持；可选另一条签名路线 |
| SideStore 找不到 provisioning profile | 签名／描述文件环节失败，不是简单“手机没信任” | 核对签名工具、App IDs、描述文件和 WDA 测试组件 |
| 剩余 App ID 只有 1 | 免费账户有 App ID 限制；删除图标不保证马上释放周内名额 | 看 View App IDs 和到期时间，别反复随机改包名 |
| 看得到，点不了 | WDA 服务、授权或测试组件可能未就绪 | 先看手机 XCTest 弹窗和触控验收，再排其他层 |
| 点按钮返回成功但数字不变 | 返回值未证明实际命中 | 查看 AssistiveTouch 圆球、系统弹窗和坐标；我们确实遇到圆球挡按钮 |
| 电脑自动切到其他 Wi‑Fi | 当前网络路径已经变了；USB、热点和 Wi‑Fi 都要区分 | 核对连接的网络名、手机地址和服务，不直接将全部归为 WDA 故障 |
| 日常能用，游戏卡顿／超时 | 自动化链路可用不等于游戏流畅；不能据此确诊内存不足 | 记录具体应用表现。本次停止了大型游戏排查 |
| AI 不回消息，但手机仍可操控 | 可能卡在 AI 会话、工具审批或消息渠道 | 单独检查 AI 任务与消息渠道，不先重装手机组件 |

免费账户的应用数和 App ID 限制见[SideStore FAQ](https://docs.sidestore.io/docs/faq)。这里记录我们的故障现象，没有将每项推测都写成最终病因。

## 六、续签：与日常操控分开

使用免费开发签名时，注意通常的 7 天有效期；以签名工具实际显示为准。续签需要相应网络和账号服务可用，不等于必须使用某一个固定 Wi‑Fi，也不等于要一直开 USB 共享网络。[SideStore FAQ](https://docs.sidestore.io/docs/faq)

如果用 SideStore，按照其[当前安装／刷新指南](https://docs.sidestore.io/zh/docs/installation/install)完成 LocalDevVPN 连接、登录和刷新。本文无线操控的实测配置在测试期间让 LocalDevVPN 保持断开；不能由此推出 LocalDevVPN 在所有设备上都会导致断线。

如果使用原项目的 Sideloadly／`fix-input` 路线，按原指南维护 WDA 的完整签名。无论哪条路线，续签后都再测一次真实点按，不只看日期更新。

## 七、给 AI 一部手机时的使用设置

用备用机开始，保留可立即停止操作的 STOP 入口，并明确哪些外部操作需要确认。不要把控制服务暴露到公网。

本次扩展保留原来的发送审批设置。但它不意味着所有“点发送”的低层点按都会自动被审批拦下；客户端指令、监督和实际入口也需要配合。手机显示的网页、聊天内容是待处理数据，不应自动获得指挥 AI 的权限。

## 八、公开分享的材料

可分享：步骤、版本、经过脱敏的错误、验收结果、原项目链接，以及整理后的改动代码。

不要上传原机 `.env`、`.state`、苹果 Lockdown 配对文件、签名证书／私钥、描述文件、完整设备日志和私人聊天。它们不属于读者复现时必须下载的素材。

分发基于 SideTap 的代码时保留它的完整 LICENSE。netmuxd、go-ios、WDA 和安装工具各自保留项目署名与许可；链接它们的官方发布页比打包未知来源的安装包更清楚。
