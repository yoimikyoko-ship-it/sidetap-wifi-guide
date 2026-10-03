# SideTap Wi‑Fi / Personal Hotspot Guide

# 把备用 iPhone 接给 AI：教程与无线热点实测

**教程整理：电脑端的 Codex 阿澈。**

面向想给 AI 伴侣一部备用手机的人，也供 AI 助手查阅步骤和故障信息。

这是独立的中文教程仓库。基础项目是 **[ucsandman / Wes Sander 的 SideTap](https://github.com/ucsandman/SideTap)**，使用 MIT 许可。本文没有官方合作或背书关系。

> 当前仓库提供教程、配置示例、测试摘要和适配设计，**尚未提供可直接安装的无线扩展代码**。只克隆原项目再复制这里的示例配置，不会自动获得本文的无线控制。

## 从哪里开始

| 你想做什么 | 阅读入口 |
| --- | --- |
| 安装基础链路、了解这次实测 | [完整教程](docs/tutorial.zh-CN.md) |
| 研究自开热点、跨机型适配和公开版改动 | [无线适配与发布说明](docs/wireless-compatibility.zh-CN.md) |
| 找可复制的小红书正文 | [正文、标题与首条评论](social/xiaohongshu-post.zh-CN.md) |
| 排 10 张图文卡片 | [图文分页稿](social/xiaohongshu-carousel.zh-CN.md) |
| 查看示例参数 | [wireless.env.example](examples/wireless.env.example) |
| 核对实测范围 | [脱敏验收摘要](evidence/acceptance-summary.json) |
| 让其他 AI 快速读懂 | [llms.txt](llms.txt) |
| 看原作者与依赖署名 | [CREDITS](CREDITS.md) |

## 无线拓扑

```text
受控 iPhone 13（蜂窝网络＋个人热点＋WDA）
                        ↕ Wi-Fi
Windows（netmuxd＋本地兼容层＋SideTap）
                        ↕ 本机 MCP
                     AI 客户端
```

实测是受控手机**自己开热点**，电脑通过 Wi‑Fi 连接它。初次安装、信任和配对仍需要数据线；日常的 Wi‑Fi 控制与 USB 共享网络、SideStore 续签是不同的环节。

## 已经成功的部分

- 计算器实际点按与读回：7＋1＝8，闲置后＋2＝10，AI 实际 MCP 入口＋3＝13。
- 放置一晚后仍能控制；这不是整夜持续进行点按测试。
- 使用者反馈：小鱼自己发 QQ、联系 GPT 均已成功。此处不发布私人聊天，也不承诺所有平台都有现成发送模块。

实测组合：iPhone 13 / iOS 26.6、Windows、go-ios 1.3.2、netmuxd 0.4.3、WDA 16.12.9。

其他手机还没完成同等验证。旧 13 玩大型游戏出现明显卡顿与超时，用户感觉手机带不动；尚未确诊具体性能或内存原因。

## 我们整理的主要坑

- 充电没有掉，不证明 USB 数据一直正常。
- WDA 有图标，不代表嵌套测试组件、签名和触控授权都正常。
- AssistiveTouch 圆球可能挡住按钮，工具成功返回也要看画面结果。
- 基础设备信息可读，不代表无线心跳与应用服务链路都正常。
- 手机可以控制，不代表 AI 会话、审批或 QQ 渠道一定在工作。

详见[完整教程的踩坑速查](docs/tutorial.zh-CN.md#五踩坑速查)。原来的 USB 故障未被唯一定位，因此不提供“全部断线都由某项省电设置导致”的结论。

## 跨机型适配状态

目标是逐步统一设备身份、地址、端口、状态目录和版本检查，并补齐不同设备验收。当前没有“全机型支持”的结论；Android 需要独立后端。

请先读[适配说明](docs/wireless-compatibility.zh-CN.md)。如果分享测试结果，只需说明设备类别、版本、拓扑和卡在哪一步；不要上传密码、验证码、设备配对记录或私人聊天。

## Credits & License

- 原项目：[ucsandman/SideTap](https://github.com/ucsandman/SideTap)，作者 Wes Sander。[原始 MIT 许可副本](third_party/SideTap-LICENSE.txt)
- 无线依赖：[jkcoxson/netmuxd](https://github.com/jkcoxson/netmuxd)。其余组件署名见 [CREDITS](CREDITS.md)。
- 本仓库教程与新增文档采用 [MIT](LICENSE)；复制时保留对应版权声明与许可。各依赖仍遵循自己的许可。

## English summary for search and AI readers

A Chinese community guide to connecting a spare iPhone to an AI companion using SideTap on Windows, with a locally tested Wi‑Fi personal hotspot transport. The controlled iPhone hosts the hotspot; the PC connects over Wi‑Fi. Initial USB pairing is still required.

This repository contains documentation and compatibility design, not a runnable wireless installer. Full wireless control was tested on one iPhone 13; other devices and Android support are not established. QQ/GPT success is user reported. Large game performance is not guaranteed. See [llms.txt](llms.txt) and the sanitized [acceptance summary](evidence/acceptance-summary.json).
