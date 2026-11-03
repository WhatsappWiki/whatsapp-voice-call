# 术语与常见问题 · Glossary and FAQ

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 术语速查

| 名称 | 含义与详细说明 |
| --- | --- |
| stanza / call-id | 单条信令／整场通话的标识，见[标识表](01-architecture.md#zh-identifiers) |
| call-creator / 设备 JID | 通话创建者／实际设备身份，见[设备身份](01-architecture.md#zh-device-identity) |
| ACK / receipt | 处理确认／接收回执，见[确认语义](02-call-lifecycle.md#zh-confirmations) |
| preaccept / accept | 设备预接受／接受通话，见[主线流程](02-call-lifecycle.md#zh) |
| destination / enc | 按设备组织目标／加密材料节点，见[设备扇出](04-device-routing-and-keys.md#zh-device-fanout) |
| encopt | 密钥生成选项，见[encopt 表](03-signaling-fields.md#zh-shared-encopt) |
| te / te2 / rte | 二进制端点列表／所选端点，见[地址协商](05-transport-negotiation.md#zh-endpoints) |
| relay | 中继信息与授权结构，见[relay 交互](05-transport-negotiation.md#zh-relay) |
| voip_settings | 通话配置 JSON 树，见[配置处理](05-transport-negotiation.md#zh-updates) |
| transaction-id | 测量、更新或结束等子流程的事务标识 |
| RTP | 实时媒体传输，见[公开 Wiki 语音目录](https://github.com/WhatsappWiki/whatsapp-technical-wiki/tree/main/calls#zh) |

### 常见问题

| 问题 | 简答与阅读入口 |
| --- | --- |
| offer ACK 表示已接通吗？ | 它确认相应信令的处理结果；接听和媒体状态另行推进。[确认语义](02-call-lifecycle.md#zh-confirmations) |
| offer receipt 表示已解密吗？ | receipt 可先于完整解密安排发送。[接收 offer](02-call-lifecycle.md#zh-confirmations) |
| preaccept 会选定最终设备吗？ | 它可更新临时活动设备信息，正式选择由对应分支决定。[设备选择](04-device-routing-and-keys.md#zh-device-selection) |
| accept 先到为什么还要等待？ | 前置上下文未齐时会缓存 Accept，再重投处理。[乱序处理](06-state-and-recovery.md#zh-accept-ordering) |
| 每条 stanza 能共用 call-id 作为 id 吗？ | 两者作用域不同，应分别标识通话与信令事务。[标识表](01-architecture.md#zh-identifiers) |
| 所有 body 都是 Base64 吗？ | 字节与文本字段各按自身规则编码。[编码说明](03-signaling-fields.md#zh-body-encoding) |
| audio 写 opus 就固定用普通 Opus 吗？ | 实际路径还受能力与配置控制，可进入 MLow。[音频字段](03-signaling-fields.md#zh-shared-audio) |
| 中继会取得语音明文吗？ | 中继转发端到端加密媒体，通话秘密由设备间会话分发。[公开 Wiki 语音目录](https://github.com/WhatsappWiki/whatsapp-technical-wiki/tree/main/calls#zh) |
| 一个设备拒绝后为何其他设备仍响铃？ | 设备身份、原因与当前状态决定处理范围。[设备选择](04-device-routing-and-keys.md#zh-device-selection) |
| terminate 后为什么又来 offer？ | 信令可能迟到，结束结果缓存负责过滤。[迟到 offer](06-state-and-recovery.md#zh-late-offer) |
| 挂断会关闭整个登录连接吗？ | 正常挂断只结束本场通话资源。[连接生命周期](01-architecture.md#zh-connection-lifecycle) |

### 公开标准与说明

- [XMPP Core](https://www.rfc-editor.org/rfc/rfc6120.html)：标准 XMPP 基础。
- [RTP](https://www.rfc-editor.org/rfc/rfc3550.html)：实时媒体传输协议。
- [WhatsApp 多设备架构](https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/)：设备扇出与通话秘密分发。
- [WhatsApp 通话隐私](https://engineering.fb.com/2023/11/08/security/whatsapp-calls-enhancing-security/)：通话中继与 IP 保护。

[English](#en)

<a id="en"></a>

## English

[Home](../README.md#en) · [中文](#zh)

### Terminology

| Term | Meaning and details |
| --- | --- |
| stanza / call-id | One signal versus the entire call; see [identifiers](01-architecture.md#en-identifiers) |
| call-creator / device JID | Call creator versus actual device identity; see [device identity](01-architecture.md#en-device-identity) |
| ACK / receipt | Processing acknowledgment versus reception receipt; see [acknowledgment semantics](02-call-lifecycle.md#en-confirmations) |
| preaccept / accept | Device preacceptance versus call acceptance; see the [main flow](02-call-lifecycle.md#en) |
| destination / enc | Per-device targets versus encrypted material; see [device fanout](04-device-routing-and-keys.md#en-device-fanout) |
| encopt | Key-generation options; see [encopt](03-signaling-fields.md#en-shared-encopt) |
| te / te2 / rte | Binary endpoint lists versus selected endpoint; see [transport negotiation](05-transport-negotiation.md#en-endpoints) |
| relay | Relay information and authorization; see [relay interaction](05-transport-negotiation.md#en-relay) |
| voip_settings | Call-configuration JSON tree; see [configuration handling](05-transport-negotiation.md#en-updates) |
| transaction-id | Transaction identifier for a measurement, update, termination or another subprocedure |
| RTP | Real-time media transport; see the [public Wiki voice directory](https://github.com/WhatsappWiki/whatsapp-technical-wiki/tree/main/calls#en) |

### Frequently asked questions

| Question | Short answer and further reading |
| --- | --- |
| Does an offer ACK mean the call is connected? | It reports processing of the signal; answering and media state advance separately. [Acknowledgment semantics](02-call-lifecycle.md#en-confirmations) |
| Does an offer receipt prove decryption? | A receipt may be scheduled before full decryption. [Receiving an offer](02-call-lifecycle.md#en-confirmations) |
| Does preaccept select the final device? | It may update temporary active-device information; formal selection follows the corresponding branch. [Device selection](04-device-routing-and-keys.md#en-device-selection) |
| Why wait when accept arrives first? | Accept is cached when prerequisite context is missing, then redispatched. [Reordering](06-state-and-recovery.md#en-accept-ordering) |
| Can every stanza use call-id as its id? | The scopes differ, so identify the call and individual signaling transactions separately. [Identifiers](01-architecture.md#en-identifiers) |
| Is every body Base64? | Byte fields and text fields follow their own encodings. [Encoding](03-signaling-fields.md#en-body-encoding) |
| Does audio@enc=opus always mean ordinary Opus? | Capabilities and configuration also select the path, which may use MLow. [Audio fields](03-signaling-fields.md#en-shared-audio) |
| Does a relay obtain plaintext audio? | It forwards end-to-end encrypted media; device sessions distribute the call secrets. [Public Wiki voice directory](https://github.com/WhatsappWiki/whatsapp-technical-wiki/tree/main/calls#en) |
| Why do other devices still ring after one rejects? | Device identity, reason and current state determine the scope. [Device selection](04-device-routing-and-keys.md#en-device-selection) |
| Why can offer arrive after terminate? | Signals may arrive late; the outcome cache filters them. [Late offers](06-state-and-recovery.md#en-late-offer) |
| Does hangup close the login connection? | Normal hangup ends only the current call's resources. [Connection lifetime](01-architecture.md#en-connection-lifecycle) |

### Public standards and explanations

- [XMPP Core](https://www.rfc-editor.org/rfc/rfc6120.html): standard XMPP foundations.
- [RTP](https://www.rfc-editor.org/rfc/rfc3550.html): real-time media transport protocol.
- [WhatsApp multi-device architecture](https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/): device fanout and call-secret distribution.
- [WhatsApp call privacy](https://engineering.fb.com/2023/11/08/security/whatsapp-calls-enhancing-security/): call relaying and IP protection.

[中文](#zh)
